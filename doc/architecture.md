# AlifePlus Architecture

AlifePlus is a reactive framework for STALKER Anomaly. It intercepts X-Ray engine callbacks, classifies them into causes through world-state predicates, and dispatches consequences through two pipelines: an **Event Pipeline** that filters engine noise into meaningful cause events, and a **Dispatch Pipeline** that routes causes to consequence handlers. The framework owns protection, rate limiting, squad ownership, lifecycle management, and structured tracing. Domain logic registers a predicate and a handler. The framework runs the pipeline.

The architecture splits into **core** (pipeline infrastructure) and **ext** (domain logic). Core never imports ext. All domain logic reaches the framework through registered function references.

Built on xlibs. See `conventions.md` for naming rules, result codes, MCM settings, logging format.

![System Overview](img/system-overview.png)

---

## Glossary

| Term | Definition |
|------|------------|
| Cause | Engine callback + world-state predicate -> meaningful event or nil |
| Consequence | Handler subscribed to a cause; executes response logic + side effects |
| Predicate | Pure function: `(trace, ...args) -> { cause, ...payload }` or nil |
| Producer | Dispatches callbacks through gate chain, evaluates predicates, publishes to xbus |
| Consumer | Receives cause events from xbus, dispatches to consequences via round-robin |
| xbus | Pub/sub event bus (xlibs). Causes publish, consequences subscribe |
| Core | Framework modules (ap_core_*). Pipeline, lifecycle, protection, rate limiting |
| Ext | Domain modules (ap_ext_*). Causes, consequences, data, messages |

---

## System Layers

AlifePlus is split into two layers: **core** and **ext**. Core is the framework - it knows nothing about massacres, stashes, or alphas. Ext is the domain - it knows nothing about gate chains, rate limiting, or squad lifecycle. The boundary is enforced by a hard rule: **core never imports ext**. All domain logic reaches the framework through registered function references (predicates, handlers, arrival callbacks).

### Core

The pipeline, protection, squad lifecycle, rate limiting, tracing, and configuration. Any cause/consequence mod can plug into core without modifying it.

| File | Role |
|------|------|
| ap_core_producer | Gate chain, predicate evaluation, xbus publish |
| ap_core_consumer | Dispatch: xbus subscribe, consequence iteration, result codes, rate gating |
| ap_core_broker | Ownership registry, squad scripting lifecycle, activity record, arrival detection, protection, PDA markers |
| ap_core_limiter | Rate limiting: cause counter, consequence token bucket, global consequence counter, cooldowns |
| ap_core_debug | Logger, observe() tracing, result helpers. Zero overhead below DEBUG |
| ap_core_utils | Event pub/sub wrappers, PDA dispatch, find_squads/find_smart with protection filters |
| ap_core_const | Enums: CALLBACK, CAUSE_TYPE, RESULT, TRACE, SQUAD_ACTION, timing constants |
| ap_core_mcm | MCM defaults, config snapshot (cfg table), UI builder |
| ap_core_compat | Save data cleanup for version upgrades, proxy ownership registrations |
| _ap_deps | Dependency gate: assert xlibs installed and version-compatible |

### Ext

Causes, consequences, domain state, messages, test tools. Ext files register with core at `on_game_start` and never touch the pipeline internals.

| File | Role |
|------|------|
| ap_ext_cause_* | Cause predicates (one file per cause group) |
| ap_ext_consequence_* | Consequence handlers (one or more files per cause) |
| ap_ext_tracker | Domain state: kill counts, alphas, stalker needs DTO |
| ap_ext_smart_mutator | Runtime smart terrain mutations (territory conquest) |
| ap_ext_object_mutator | Runtime combat modifiers for alpha mutants (hit power scaling, panic immunity) and high-rank stalkers (rank-based hit power) |
| ap_ext_util | Domain gates: alignment, personality checks, FIFO-cached species resolution |
| ap_ext_const | Community sets, faction lists, item pools |
| ap_ext_messages | PDA message templates |
| ap_ext_test | In-game debug commands |

---

## Core

### Execution Model

X-Ray runs Lua on a single thread. There is no concurrency within one engine tick. When the engine fires a callback like `squad_on_update`, the entire AP pipeline executes synchronously in one Lua call stack before control returns to the engine. The producer evaluates gates, the cause predicate runs, xbus publishes, the consumer iterates all consequences, each handler runs and may script a squad. All of this completes inside a single function call.

```
engine squad_on_update
  -> producer._on_radiant (gate chain: pacer_1, is_protected, ratio, pacer_2)
    -> cause cascade: try each predicate starting at cursor, stop on first publish
      -> cause predicate (ext) returns nil -> try next cause
      -> cause predicate (ext) returns { cause = CAUSE.X, ...payload } -> publish + break
        -> _try_publish -> xbus.publish (synchronous, inline)
          -> consumer._process (iterates all consequences for this cause)
            -> consequence handler (ext) returns { code = RESULT.X }
              -> script_squad (core) sets scripted_target, registers arrival
          <- all consequences done, returns to producer
        <- cascade breaks on first publish
```

When `_try_publish` returns, the consequences have already executed, squads have been scripted, and counters have been incremented. `xbus.publish` calls each subscriber function inline and returns when all have finished.

### Event Pipeline

The producer receives engine callbacks and filters them through a chain of gates before evaluating cause predicates. Two variants exist because radiant and reactive events have fundamentally different characteristics.

**Radiant events** are ambient observations. A squad periodically scans its surroundings and notices something (a stash, empty territory, an unmet need). The squad is both sensor and responder. High frequency, low significance per event.

**Reactive events** are world-state changes. Something happened (a death, a healing, a pickup) and the framework reacts. The triggering entity may not be the entity AP acts on. Low frequency, high significance per event.

#### Radiant variant

Radiant causes fire on `squad_on_update` - the only stable, uniform heartbeat covering both online and offline squads. The engine fires this callback from `sim_squad_scripted:update()` for every squad every tick. Online squads fire at frame rate (~3/sec per squad), offline squads fire via the A-Life scheduler round-robin (~0.03/sec per squad, ~20 squads per tick across ~776 total). This produces a raw volume of ~6,600 calls/min that the gate chain must reduce to a manageable rate.

Radiant causes are an open collection. New causes can be added by registering a predicate on `RADIANT_CALLBACKS`. Each admitted call cascades through all registered radiant causes starting at a round-robin cursor. The first cause whose predicate publishes stops the cascade. If all predicates reject, nothing publishes. The cursor advances past the last-tried cause, so each admission starts at a different cause. With `distributor_interval_sec = 5s` (~12 triggers/min), every admission tries up to 4 causes (worst case). This is different from reactive causes, where all causes for a callback evaluate independently on every admitted event.

The triggering squad is both sensor and responder - it evaluates its own surroundings and acts on what it finds (stashes, empty territory, unmet needs).

**Gate 1: PACER_1.** Coarse rate limiter. Pure `os.clock()` timestamp comparison with a 100ms interval (~10 admits/sec). Runs before any squad field access. No `squad.id`, no luabind. Rejects ~98% of raw `squad_on_update` volume at near-zero cost. Instead of caching known-bad squads (old BANNED gate), PACER_1 simply limits how many squads reach the eligibility check.

**Gate 2: is_protected.** Full eligibility check via `ap_core_broker.is_protected`. Runs on ~10 squads/sec from PACER_1. Checks ownership (warfare, BAO), is_scripted (engine `scripted_target`, condlist, random_targets), permanent (story, trader, named NPC -- session-lifetime cache), active role (task giver, companion), and task target (assault, bounty, delivery). Short-circuits early: scripted and permanent squads are caught in the first two checks. Subsumes the old BANNED and SCRIPTED gates. At 10/sec, the cost is acceptable without negative caching.

Protection is not a single gate - it is applied at four layers across the pipeline. The same guard set is checked at each layer, but against different entities depending on context:

| Layer | Where | What is checked |
|-------|-------|-----------------|
| Producer | Radiant gate 2 (is_protected) | Triggering squad, before any cause evaluates |
| Cause | Reactive predicates | The entity AP would act on (killer, patient, taker) |
| Consequence | `ap_core_utils.find_squads` | Every candidate responder squad |
| Squad | `ap_core_broker.script_squad` | No inline check - protection is upstream |

Reactive causes skip the producer protection gate because the callback entity (e.g. a dead victim in `squad_on_npc_death`) is not the entity AP would script. Instead, reactive causes check protection on the relevant entity inside the predicate itself: `alpha` and `alphakill` guard the killer, `wounded` guards the patient, `harvest` guards the taker. Causes where the trigger is a dead victim (`massacre`, `squadkill`, `basekill`) need no guard - consequences find responders through `find_squads` which applies all exclusions.

`script_squad` does not check protection. It assumes all upstream layers have already verified the squad. Direct callers outside the pipeline must check `is_protected` themselves.

**Gate 3: RATIO.** Bresenham integer admission gate. Off-map events outnumber on-map ~50-100:1 per squad because online squads fire at frame rate while offline squads fire via scheduler round-robin. The ratio gate restores balance using integer cross-multiplication: `throttled_count * |r| <= (10 - |r|) * favored_count`. At the default ratio of 8, this admits ~4 on-map events per 1 off-map. The `squad.online` field (C++ `m_bOnline`, refreshed by `check_online_status()` immediately before the callback) determines on-map status with zero luabind cost. Radiant and reactive pipelines maintain separate counter pairs (`_radiant_ct`, `_reactive_ct`) so high-volume radiant traffic does not bias reactive admission. Counters reset at 32768 to prevent overflow. Must be after is_protected so it only balances eligible squads.

**Gate 4: PACER_2.** Budget limiter. `os.clock()` timestamp comparison with `cfg.distributor_interval_sec` (default 5s, ~12 triggers/min). Only fully eligible, ratio-balanced squads consume triggers. Every PACER_2 admit produces an EVAL -- zero waste. This is the key improvement over the old pipeline: the old single pacer ran before eligibility checks, so ineligible squads (scripted dogs, story NPCs) consumed triggers and starved the pipeline.

**EVAL (cascade).** Cascades through all registered cause predicates starting at the round-robin cursor (indexed array, deterministic order). Each predicate is checked against a per-cause sliding window rate limit (`cfg.cause_max_events`, 60s window) and skipped if exhausted. The first predicate to publish stops the cascade. If all predicates reject, nothing publishes. The cursor tracks the last-tried cause; next admission starts at cursor+1. Each predicate is wrapped in `observe()` for tracing. On publish, xbus dispatch is synchronous: the consumer runs all consequences inline before control returns to the producer (see Execution Model). Worst case: 4 predicate evaluations per admission (0.00-0.08ms each). At 12 admissions/min = ~1ms/min extra.

#### Reactive variant

Reactive causes fire on world events: `squad_on_npc_death`, `x_npc_medkit_use`, `actor_on_item_take`, `npc_on_item_take`, `actor_on_item_use`. These are low-frequency, high-significance events (a death, a healing, a pickup). The gate chain is shorter because the triggering entity may be a dead victim or an uninvolved bystander - protection must happen downstream in causes and consequences where the actual responder is known.

**Gate 1: PACER.** Token bucket with per-callback-type keys, 1 token/sec per type. Independent from the radiant pacer.

**Gate 2: RATIO.** Same Bresenham algorithm as radiant but with its own counter pair (`_reactive_ct`). Some reactive callbacks (`actor_on_item_use`, `actor_on_item_take`) are always on-map (they require a game_object which only exists online).

**Gate 3: EVAL.** Iterates all registered reactive cause predicates for the callback type in deterministic order (indexed array). Each predicate is checked against the per-cause sliding window rate limit, then evaluated. All causes evaluate independently. A single death event can trigger MASSACRE, SQUADKILL, and ALPHA simultaneously because different responder squads act independently. Radiant cascade stops on first publish because the triggering squad can only do one thing.

#### Instrumentation

Both pipelines measure gate span and eval span. Gate span covers the time from first eligibility check to budget admission (radiant: is_protected + RATIO + PACER_2; reactive: PACER + RATIO). Eval span covers cause predicate evaluation, xbus dispatch, and all consequence handlers. Spans are accumulated per `PACER_LOG_INTERVAL` (60s) and reported as avg/max in the periodic `[PIPELINE]` dump. All timing uses `os.clock()` and is gated by `ap_core_debug.enabled()` -- zero overhead when log level is above DEBUG.

### Dispatch Pipeline

After a cause publishes to xbus, the consumer receives the event and iterates all registered consequence handlers for that cause type.

**Iteration order.** Each cause type maintains a round-robin cursor. The cursor rotates which consequence gets tried first, distributing evaluations evenly across handlers. The consumer always tries every consequence for the cause. Consequences do not control flow. The only thing that stops the loop early is global radiant budget exhaustion.

**Before each handler runs, the consumer checks two pre-gates.** The `condition` function (registered at init, typically checks MCM enabled flag) determines if the consequence is active. If it returns false, the consumer skips the consequence and moves to the next one. Then the rate limiters are checked: if the per-type budget for this consequence is exhausted, the consumer skips it. If the global radiant budget is exhausted, the consumer stops the entire loop. The handler never runs when a pre-gate rejects.

**The handler runs and returns `{ code = RESULT.X, reason = "..." }`.** The result code tells the consumer which phase of the handler answered. Every consequence handler follows a three-phase structure called the consequence template (see Consequence Template below). `FAILED_RULES` means the handler checked its business rules (alignment, personality, species, match, validation) and rejected the event. `FAILED_EVAL` means the rules passed but a world query found nothing (find_squads returned empty, find_smart found no match, an entity lookup returned nil). `FAILED_ACTION` means the rules and eval passed but the action failed (script_squad could not route the squad, a registration call returned false). `SUCCESS` means the consequence executed its action.

**On `SUCCESS`, the consumer increments the per-type counter, increments the global radiant counter (for radiant events only), and sets `event_data._fired = true`.** This flag signals the producer that at least one consequence acted on the event. On any other result code, the consumer records stats but takes no counter action. The loop continues to the next consequence regardless of the result.

### Rate Limiting

All rate limiting lives in `ap_core_limiter`. Five independent limiters operate at different scopes.

| Limiter | Mechanism | Scope | Default | Config |
|---------|-----------|-------|---------|--------|
| Radiant pacer | os.clock timestamp | global | 5s | MCM distributor_interval_sec |
| Reactive pacer | token bucket | per-callback-type | 1/sec | constant |
| Cause budget | TTL counter, sliding window | per-cause | 10/60s | MCM cause_max_events |
| Consequence budget | token bucket (peek/acquire) | per-consequence | 2/60s | MCM consequence_max_events |
| Global consequence | TTL counter | radiant only | 5/60s | MCM global_consequence_max_events |

### Tracing

Hierarchical tracing via `observe()` in `ap_core_debug`. Each trace carries a monotonic **tid** (trace ID) and a slash-separated **path** (span hierarchy). A single `tid` links a cause through its consequence chain into individual actions:

```
[CAUSE.NEEDS] [tid=42 path=CAUSE.NEEDS] ok id=1337 need=hunger drive=4.2 [0.15ms]
[CONSEQUENCE.HUNGER_CAMPFIRE] [tid=42 path=CAUSE.NEEDS/CONSEQUENCE.HUNGER_CAMPFIRE] success count=1 [0.83ms]
[ACTION.FIND_DESTINATION] [tid=42 path=CAUSE.NEEDS/CONSEQUENCE.HUNGER_CAMPFIRE/ACTION.FIND_DESTINATION] ok id=445 [0.12ms]
```

Below DEBUG log level: `observe()` is a bare passthrough (calls the function, returns the result). `trace()` returns a null singleton. Cost: one `enabled()` check (~150ns) per call. The `_null_trace` and associated no-ops are pre-allocated singletons - no allocation at non-debug levels.

### Squad Lifecycle

`ap_core_broker` manages the full lifecycle of squads under AP control: scripting, activity recording, arrival detection, post-arrival wait, and release.

**Scripting.** `script_squad(squad, smart, opts)` sets `scripted_target` via `xsquad.control_squad`. `scripted_target` routes the squad to `specific_update` (direct A->B movement). AP clears `__lock` on acquisition -- `scripted_target` alone is sufficient for routing. If another mod clears `scripted_target` between ticks, `generic_update` runs and the squad may be reassigned by SIMBOARD; `reassert_target` restores `scripted_target` within 20s. The squad is registered in `_ap_scripted_squads` with a TTL, optional arrival handler, and wait duration. `script_actor_target(squad)` scripts a squad to pursue the player using engine-native actor targeting (no arrival detection).

**Scripted squad scan.** `_update_scripted_squads` runs every 20s via `CreateTimeEvent`. For each tracked squad:

1. **Entity** - `xobject.se(squad_id)`. Gone -> remove from tracking table. Catches squads that died or despawned between scans.
2. **Reassert** - `xsquad.reassert_target(squad, data.scripted_target)`. Restores `scripted_target` if another mod overwrote it, clears `__lock`. Every alife overhaul mod (warfare, BAO, Vintar) sets these fields every tick on its own squads; AP reasserts every 20s on its squads.
3. **TTL** - 7200 game-seconds. Expired -> unscript. Prevents permanently pinned squads.
4. **Arrival** - `xsmart.is_arrived(squad, smart)`. On arrival: if smart is full and squad is online, fire the arrival handler then unscript (overflow). Otherwise, dispatch the registered `on_arrive` function, then enter wait state.
5. **Wait** - `release_at = game_sec() + pre_release_gulag` (default 300s). When game time exceeds `release_at`, unscript. Game time advances during sleep/time-skip and survives save/load.

**Save/load.** `_ap_scripted_squads` persists to `m_data.ap_core_broker`. Engine-side `scripted_target` persists natively across save/load (`sim_squad_scripted` STATE_Write/STATE_Read). Arrival handler functions are transient - they are re-registered every load via `consumer.register` opts. On load, squads marked as arrived get `release_at = 0` (immediate release on next scan).

**Coordination.** `scripted_target` is the squad control field. Setting it routes the squad to `specific_update`. AP no longer sets `__lock` (clears it on acquisition). `scripted_target` alone is sufficient for routing; `__lock` was a redundant fallback guard. AP checks `scripted_target` at gate 2 (is_protected, via `xsquad.is_scripted`). Two alife mods that both check `scripted_target` before claiming a squad will not conflict. `xsquad.control_squad` sets `scripted_target` on acquire; `xsquad.release_squad` clears it on release; `xsquad.reassert_target` restores it if overwritten. The ownership registry (`register_owner`) adds identity on top: it tells AP WHO owns a squad, not just that it's owned. Warfare and BAO are registered by default in `ap_core_compat`.

**Markers.** Markers are derived from `_ap_record`. A 10s timer (`_apply_markers`) iterates record entries, resolves labels from `CONSEQUENCE_DESCRIPTION`, and calls `xpda.mark_squad`. A 20s timer (`_validate_markers`) removes marks for dead entities or unscripted non-persistent entries. The `persistent` flag (set via `record(id, cause, consequence, { persistent = true })`) keeps a marker until the entity dies regardless of scripted status (used by `alpha_promote`).

### Initialization Lifecycle

Four phases. Each requires the previous to complete.

**Phase 0: Module load.** The engine's auto-load mechanism (`_G.__index = auto_load` in `script_engine.cpp:375`) resolves `.script` files on first namespace access. `axr_main.on_game_start()` iterates all `.script` files in alphabetical order, triggering auto-load for each. Module-level code runs immediately on load: engine globals are cached to locals, constant tables are built, xlibs API references are captured. No game state exists at this point - no actor, no entities, no callbacks active.

**Phase 1: on_game_start.** `axr_main` calls `on_game_start()` on every loaded script. File order is alphabetical, but all operations are independent - no cross-module reads at this phase:

- `_ap_deps` asserts xlibs compatibility. Hard crash on mismatch.
- `ap_core_mcm` loads config from defaults, registers `on_option_change`.
- `ap_core_debug` registers `actor_on_first_update` for deferred log level init.
- `ap_core_producer` resets dispatch state, registers `actor_on_first_update`. Does NOT subscribe to callbacks yet.
- `ap_core_consumer` registers `actor_on_first_update`. Does NOT subscribe to xbus yet.
- `ap_core_broker` registers save/load/first_update callbacks, creates the 20s scripted squad scan timer.
- `ap_core_compat` registers `load_state` for save cleanup, registers ownership proxies (warfare, bao).
- `ap_ext_cause_*` register predicates with producer via `register()`.
- `ap_ext_consequence_*` register handlers with consumer via `register()`. Arrival handlers registered via consumer opts.

After phase 1: all predicates and handlers are registered, but no callbacks are subscribed and no xbus subscriptions are active. The deferred init pattern avoids alphabetical ordering bugs - `ap_core_producer` (alphabetically before cause files) cannot build its radiant handler set until all causes have registered, so it defers to `actor_on_first_update`.

**Phase 2: actor_on_first_update.** Game world and actor exist. Deferred initialization runs:

- Producer rebuilds the radiant handler set from all registrations, subscribes to engine callbacks.
- Consumer subscribes to xbus cause events.
- Debug reads MCM log level (config is now loaded), dumps all MCM settings at DEBUG.
- Squad clears stale map markers from previous session, starts marker timer if MCM `map_markers` is enabled.

The `actor_on_first_update` callback fires on the first frame with a live actor. It fires on every level transition (not just game start), so all handlers use a `_subscribed` guard to prevent duplicate registration. Named function references registered via `RegisterScriptCallback` are naturally deduplicated (the function reference is the table key), but the guard prevents redundant work.

**Phase 3: on_game_load.** Fires after `STATE_Read` rebuilds all server entities from the save file. At this point `alife_object(id)` works and entity mutation is safe. The smart mutator re-applies conquered smart terrain data from `_conquered_pending` (two-phase restore: `load_state` reads the data, `on_game_load` applies the mutations, because `load_state` fires before entities exist).

Critical timing from the engine load sequence:
1. `load_state` - save data read (`m_data` available), entities do NOT exist yet
2. `STATE_Read` - engine deserializes all server entities, rebuilds smart terrain config from LTX
3. `on_game_load` - entities exist, mutations can be applied

---

## Ext

### Tracker (ap_ext_tracker)

Domain state manager. Tracks kill counts per entity (`_ap_killers`), alpha status with level/kills/name (`_ap_alphas`), alpha death grace period (`_ap_alpha_dead`, xttltable TTL, 3600s), and stalker needs DTO (`_ap_stalker_needs`, per-squad timestamps for 9 drives). Only mutants become alphas. Stalker rank is handled natively by the engine. Registers the `squad_on_npc_death` handler for kill/alpha tracking. Save/load to `m_data.ap_ext_tracker`.

### Smart Mutator (ap_ext_smart_mutator)

Runtime smart terrain mutations for territory conquest. Both stalker and mutant factions can conquer smart terrains.

**Mechanism.** `conquer_smart(smart_id, faction)` calls `xsmart.set_faction_controlled` which writes `respawn_params` entries for all 19 factions (12 stalker + 7 mutant) and sets `smart.faction` to the conqueror. The engine's `try_respawn` (smart_terrain.script:1665) only spawns entries where `v.faction == self.faction`, so only the conqueror's squads appear. Stalker factions spawn `*_sim_squad_novice/advanced/veteran` sections. Mutant factions spawn `simulation_*` sections mapped by species (e.g. `monster_predatory_day` spawns dogs and pseudodogs).

**Volatility.** The engine rebuilds smart config from LTX on every load (`STATE_Read`), so the mutator uses two-phase restore (`load_state` reads to pending, `on_game_load` re-applies).

**Monster faction revert.** `check_smart_faction` (smart_terrain.script:1209) only counts `IsStalker` NPCs. When only monsters occupy an online smart, `smart.faction` reverts to `default_faction`. A 60s periodic scanner (`_scan_conquered`) re-applies `smart.faction` for monster-conquered smarts.

**Decay.** Conquered smarts revert to their original faction after `cfg.area_conquest_decay_hours` game hours (default 48). The scanner checks `xtime.game_sec() - conquered_at` and calls `clear_faction_controlled` on expired entries. Decay creates a natural territorial cycle: factions conquer, territory decays, other factions reconquer.

**Eviction.** FIFO at `cfg.area_conquest_max_smarts` (default 50). Oldest conquest by game time is evicted when the cap is reached. Same-faction re-conquest refreshes the timestamp (LRU). Different-faction overwrites without eviction.

### Object Mutator (ap_ext_object_mutator)

Runtime combat modifiers for alpha mutants and high-rank stalkers. Two independent systems on `monster_on_before_hit` and `npc_on_before_hit`.

**Alpha mutants** (`monster_on_before_hit`): outgoing hit power bonus and incoming hit power absorption via `_alpha_hit_power_dealt[npc_id]` / `_alpha_hit_power_taken[npc_id]` hash tables, populated at promote time. Panic immunity (`set_custom_panic_threshold(0)`) applied lazily on first hit. O(1) lookup, 0.5s throttle, early exit when tables empty. Alpha level: `min(10, floor(kills / alpha_kills_per_level))`. Loot items injected via `monster_on_loot_init` callback with per-species bonus pools, managed in `ap_ext_tracker`.

**Stalker rank** (`npc_on_before_hit`): outgoing hit power bonus and incoming hit power reduction for veteran+ stalkers (rank 12000+). Linear scaling from veteran to legend. Reads engine `character_rank()`, never manipulates it.

### Causes

| Cause | Type | Callback(s) | Conditions |
|-------|------|-------------|------------|
| MASSACRE | reactive | squad_on_npc_death | IsStalker(victim), at smart, not is_base, kill_count >= threshold |
| SQUADKILL | reactive | squad_on_npc_death | last member dead, not is_base |
| BASEKILL | reactive | squad_on_npc_death | IsStalker(victim), at is_base, kill_count >= threshold |
| ALPHA | reactive | squad_on_npc_death | killer is mutant, not protected, projected kills cross new level |
| ALPHAKILL | reactive | squad_on_npc_death | victim is alpha, killer not protected, cooldown clear |
| WOUNDED | reactive | x_npc_medkit_use, actor_on_item_use | subject not protected, not is_base |
| HARVEST | reactive | actor_on_item_take, npc_on_item_take | IsArtefact(item), NPC taker not protected |
| STASH | radiant | squad_on_update | not protected, stash within 500m (no cause-level alignment filter, consequences handle it) |
| AREA | radiant | squad_on_update | not protected, empty smart, not is_base |
| NEEDS | radiant | squad_on_update | not protected, alignment_human, level.present(), Hull drive scoring |
| INSTINCTS | radiant | squad_on_update | not protected, alignment_mutant, species resolved, Hull drive scoring |

Predicate contract: `function(trace, ...callback_args) -> { cause = CAUSE.X, ...payload } | nil`. Producer wraps each call in `observe()`, attaches `._trace`, publishes to xbus, increments cause counter. Predicates only evaluate and return - no observe(), publish(), trace creation, or counter manipulation.

#### Cause Classification

All causes are reactive. The framework reacts to something. The pipeline splits into two mechanisms:

- **Simple (reactive pipeline):** a discrete world action triggers the cause. A death, a healing, a pickup. The squad did not ask for it. Engine callback fires, producer evaluates all registered predicates for that callback type.
- **Radiant (radiant pipeline):** the squad evaluates its own state on its update tick. Nothing specific happened. The framework checks what the squad sees, feels, or senses.

Radiant causes split into three behavioral categories based on what the squad evaluates:

| Category | Causes | What drives behavior | Range |
|----------|--------|---------------------|-------|
| Reactions | MASSACRE, SQUADKILL, BASEKILL, ALPHA, ALPHAKILL, WOUNDED, HARVEST | World event triggers response | signal (stalkers), scent (mutants) |
| Opportunities | STASH, AREA | Squad sees what's nearby | eye |
| Needs | NEEDS | Stalker internal drives, scored by deprivation | signal |
| Instincts | INSTINCTS | Mutant internal drives, scored by deprivation | scent |

Reactions are simple-mechanism causes. Opportunities, Needs, and Instincts are radiant-mechanism causes.

### Consequences

| Cause | Consequence | Actor | Effect |
|-------|-------------|-------|--------|
| MASSACRE | massacre_investigate | stalker | victim faction investigates |
| MASSACRE | massacre_scavenge | mutant | cowardly mutants converge on massacre site |
| SQUADKILL | squadkill_revenge | stalker | victim faction pursues killer (chase) |
| SQUADKILL | squadkill_flee | stalker | victim faction retreats to base |
| BASEKILL | basekill_support | stalker | friendly squads reinforce attacked base |
| BASEKILL | basekill_flee | stalker | squads at base evacuate |
| ALPHA | alpha_promote | mutant | update alpha level, apply hit power/panic buffs, grant loot |
| ALPHAKILL | alphakill_targeted | mutant | other alpha mutants on same level pursue killer (chase) |
| WOUNDED | wounded_hunt | mutant | predator+aberrant mutants close in |
| WOUNDED | wounded_help | stalker | same-faction squads rush to help |
| HARVEST | harvest_rob | stalker | outlaws pursue artefact taker (chase) |
| HARVEST | harvest_haunt | mutant | aberrant mutants converge on pickup site |
| STASH | stash_loot | stalker | loot stash to NPC inventory (CONFIGS-driven) |
| STASH | stash_ambush | stalker | camp at stash, passive (CONFIGS-driven) |
| STASH | stash_fill | stalker | fill stash with items (CONFIGS-driven) |
| AREA | area_conquer | both | claim empty smart terrain (stalkers and mutants, decays after 48h) |
| NEEDS | (14 entries) | stalker | hunger, sleep, rest, heal, shelter, money, supply, job, social (CONFIGS-driven) |
| INSTINCTS | instincts_feed | mutant (all) | move to territory to hunt/scavenge |
| INSTINCTS | instincts_sleep | mutant (all) | return to species-appropriate rest location (cowardly->territory, feral->lair, predator->lair/surge, aberrant->surge) |
| INSTINCTS | instincts_explore | mutant (cowardly+feral+predator) | wander to nearby territory or lair. Aberrant excluded (lair-bound). |
| INSTINCTS | instincts_socialize | mutant (cowardly+feral) | move toward smart with same-faction squads. Predator and aberrant excluded (solitary). |

Handler contract: `function(event_data) -> { code = RESULT.X, reason = "..." }`. All domain gates (alignment, species, personality) live in ext consequence code, never in core. Dispatch order: round-robin cursor per cause type.

### Consequence Template

Every consequence handler follows a fixed three-phase structure (Template Method):

```
rules -> eval -> action
```

| Phase | What runs | Returns on fail |
|-------|-----------|-----------------|
| rules | alignment, species (direct hash), personality, match (need/instinct), validation (nil args, at_base) | `FAILED_RULES` |
| eval | find_squads, find_smart, xobject.se (entity lookup) | `FAILED_EVAL` |
| action | script_squad, script_actor_target, update_alpha, conquer_smart | `FAILED_ACTION` |
| action (ok) | | `SUCCESS` |

Each phase runs in order. If a phase fails, the handler returns immediately with the corresponding result code. The handler never reaches a later phase if an earlier phase failed.

Gate order within the rules phase: alignment -> species (direct hash on resolved identity) -> personality -> match -> validation.

### Domain Gates

Alignment and personality are domain-level gates. They live exclusively in ext consequence code (`ap_ext_util`), never in core. Core knows nothing about factions, species, or personality traits. Species filtering is a direct hash lookup on the resolved identity, not a separate function.

**Gate order inside every consequence handler:**

1. **Alignment** (hard filter, deterministic). Can this faction participate at all? Static hash set lookup on engine faction (`squad.player_id`), O(1). Checked via `ap_ext_util.check_alignment`.
2. **Species** (hard filter, deterministic). For mutants: is this the right kind of mutant? Species is resolved once via `ap_ext_util.get_species` (FIFO-cached, 0 luabind on hit). Stalkers have nil species and pass automatically. Mutants are checked via direct hash lookup on the species alignment set.
3. **Personality** (probability, non-deterministic). How likely is this faction/species to act? Averages relevant traits (max 2 per consequence), clamps to MCM `personality_min`/`personality_max` (defaults 0.20/0.70), rolls. INV_ traits resolve as `1 - base_value` before averaging. Checked via `ap_ext_util.check_personality`. The identity key (species for mutants, community for stalkers) is resolved once and shared with the species gate.

Not every consequence uses all three gates. Human-only consequences skip species. Some consequences (alpha_promote) have no gates beyond enabled check. But when gates are present, the order is always alignment -> species -> personality.

**Where gates apply depends on cause type:**

- **Radiant:** the ticking squad is both sensor and responder. Gates check `event_data.community` (the squad's faction) and `event_data.squad_id` (for species).
- **Reactive, same-faction:** gates check the event faction (victim faction, wounded faction). Responders are found via `find_squads` with that faction.
- **Reactive, cross-faction:** the alignment set IS the `factions` parameter to `find_squads`. Species and personality are checked per-squad on the found candidates, inside the find loop.

### Alignment

Hard filter: can this faction do this consequence at all. Static hash sets in `ap_ext_const`, O(1) per check. Zombied is excluded from alignment_human and therefore from all human consequences globally.

Human factions follow GSC's moral axis (Ai.doc:65-76, тип характера):

| Table | Factions | GSC origin |
|-------|----------|------------|
| alignment_principled | dolg, army, monolith, isg | Principled: follows rules, organized |
| alignment_selfserving | stalker, csky, ecolog, freedom, killer | Self-serving: independent, own goals |
| alignment_unprincipled | stalker, freedom, killer, csky | Chaotic neutral: own rules, not criminal |
| alignment_outlaw | bandit, renegade, greh | Chaotic evil: criminals |
| alignment_human | all 12, no zombied | Union of principled + selfserving + outlaw |
| alignment_naturalist | stalker, csky, freedom, ecolog | AP subset: zone dwellers |

Mutant species alignments follow GSC's creature groups (monstry.doc:4). Keyed by species string (`xcreature.get_mutant_species`), not engine faction:

| Table | Species | GSC origin |
|-------|---------|------------|
| alignment_mutant | all 7 monster factions | Fast gate for find_squads (engine player_id) |
| alignment_mutant_cowardly | flesh, zombie, tushkano, rat, karlik | Timid, flees danger, bottom of food chain |
| alignment_mutant_feral | dog, pseudodog, boar, snork, cat, gigant | Pack/herd, reactive aggression, brute apex |
| alignment_mutant_predator | lurker, bloodsucker, psysucker, chimera, fracture | Solitary hunters, ambush, pursue wounded |
| alignment_mutant_aberrant | controller, burer, poltergeist, psy_dog | Psychic, lair-bound, supernatural |

Activity alignment axis (independent of behavioral axis, gates day/night cycle):

| Table | Species | GSC origin |
|-------|---------|------------|
| alignment_mutant_night | bloodsucker, psysucker, lurker, chimera, zombie, fracture | monstry.doc: bloodsucker "noch'yu vykhodit na otkrytyye mesta, okhotit'sya". Engine: monster_predatory_night, monster_zombied_night factions. |
| alignment_mutant_day | all other species | Engine: monster_predatory_day, monster_zombied_day. Default for lair-bound species. |

Consequences compose tables with `xtable.merge` (set union) and `xtable.subtract` (set difference) at module load.

### Personality

Probability layer: how likely is an eligible faction/species to act. Runs only after alignment passes. All checks happen in ext consequence code via `ap_ext_util.check_personality`, never in core.

Stalker factions have 7 traits: aggression, greed, survival, perception, territory, relation, discipline. Mutant species have 5 traits: aggression, survival, territory, perception, relation. Each consequence declares at most 2 relevant traits. The check averages those traits for the faction/species and rolls `math.random()` against the result, clamped to MCM `personality_min`/`personality_max`.

**Formula:** `chance = clamp(avg(relevant_traits), p_min, p_max)`. No per-consequence weight. Each consequence defines its own floor (`p_min`) and ceiling (`p_max`) as code-level constants (defaults: 0.20/0.70). The floor ensures even unfavorable factions act occasionally. The ceiling ensures even favorable factions fail sometimes. Per-consequence ownership allows individual tuning without a global lever affecting all behaviors.

**Inverted traits:** traits prefixed with `INV_` resolve as `1 - base_value` before averaging. Used for behaviors driven by the absence of a quality: fleeing is gated by `INV_DISCIPLINE` + `INV_TERRITORY` (low discipline and low territorial attachment = more likely to flee). Only `INV_DISCIPLINE`, `INV_TERRITORY`, and `INV_AGGRESSION` are defined.

**Trait value design:** all traits use tiered values in 0.10 steps (0.10, 0.20, ..., 0.90) to ensure clean math. Survival is a flat band (0.40-0.60) for all factions and species. Biological needs are universal drives, not faction differentiators. Each value is a direct probability grounded in GSC lore (monstry.doc, Ai.doc).

### Range Tiers

Every consequence searches within a range that matches the squad's awareness. Two tiers, grounded in GSC's `PersonalEyeRange` (EFC design docs, circa 2002) and validated against empirical smart terrain spacing (14 levels measured via TestZone, see `doc/library/modding/level-geometry.md`).

| Tier | Constant | Distance | Who |
|------|----------|----------|-----|
| EyeRange | `RANGE_EYE` | 200m | all (line-of-sight) |
| SignalRange | `RANGE_SIGNAL` | 500m | stalkers (PDA/radio) |
| ScentRange | `RANGE_SCENT` | 500m | mutants (scent tracking) |

EyeRange covers the p90 nearest-neighbor distance on 13 of 14 measured levels. A squad at any smart terrain can see 1-3 neighboring smarts and several stashes within 200m. SignalRange and ScentRange cover the full operational radius: stalkers receive PDA alerts, mutants track scent. Same distance today (500m), independently tunable.

Each cause type maps to a range based on how the squad learns about the opportunity:

| Cause type | Stalker range | Mutant range | Rationale |
|------------|---------------|--------------|-----------|
| Opportunities (stash, territory) | EyeRange | EyeRange | Squad sees what's nearby on arrival |
| Reactions (kills, massacres, wounded) | SignalRange | ScentRange | Stalkers hear over radio, mutants smell blood |
| Needs (hunger, sleep, shelter) | SignalRange | - | Squad knows campfire/trader locations from PDA |
| Instincts (feed, sleep, explore) | - | ScentRange | Mutants track by scent across the area |

Three tiers create a natural separation: opportunities are local and opportunistic (200m), stalker coordination reaches further via radio (500m), and mutant hunting extends through scent (500m). You act on what you see, respond to what you hear, hunt what you smell.

### Day/Night Cycle

All drives (stalker needs and mutant instincts) are gated by the active/dormant period system. Each drive entry declares `active_period = true` (fires only when the creature is active) or `dormant_period = true` (fires only when dormant). Drives with neither flag fire at any time.

`ap_ext_util.is_active_period(identity)` resolves the current period for any species or community. Nocturnal species (`alignment_mutant_night`) are active at night (20:00-05:00). All others (stalkers, diurnal mutants) are active during day (05:00-20:00).

**Stalker needs:**

| Drive | Flag | Effect |
|-------|------|--------|
| hunger | (none) | any time |
| sleep | dormant_period | night only |
| rest | dormant_period | night only |
| heal | (none) | any time |
| shelter | dormant_period | night only |
| supply | active_period | day only |
| money | active_period | day only |
| job | active_period | day only |
| social | (none) | any time |

**Mutant instincts:**

| Drive | Flag | Effect |
|-------|------|--------|
| feed | active_period | active period only |
| sleep | dormant_period | dormant period only |
| explore | active_period | active period only |
| socialize | active_period | active period only |

### Invariants

1. **Serialization boundary.** `m_data` accepts only primitives and tables of primitives. Functions, userdata, metatables are silently dropped on save. Arrival handler keys are strings; handler functions are transient, re-registered every load.

2. **Result code contract.** Every consequence returns `{ code = RESULT.X }` where X is one of `SUCCESS`, `FAILED_RULES`, `FAILED_EVAL`, `FAILED_ACTION`. Consumer reads `.code` for chain control and counter updates. Missing or malformed return = error.

3. **Scripted bypass.** `scripted_target` overrides `SIMBOARD:get_squad_target()`. Engine `target_precondition` (faction check, capacity check) is NOT evaluated for scripted squads. AP enforces faction safety in consequence filters. Capacity is handled by arrival overflow.

4. **Mutation volatility.** Runtime smart terrain mutations are rebuilt from LTX on every load. The smart mutator saves and restores independently using two-phase restore. A 60s periodic scanner re-applies monster faction ownership (engine's `check_smart_faction` ignores monsters) and decays expired conquests.

5. **Core/ext boundary.** Core never imports ext. All domain logic reaches core through registered function references.

6. **Stalker/mutant consequence separation.** When stalkers and mutants respond to the same cause, they get separate consequences. A massacre triggers `massacre_investigate` (stalkers) and `massacre_scavenge` (mutants) independently. Separation exists because the behaviors differ (stalkers investigate, mutants feed) and because simultaneous responses create emergent interactions (stalkers arrive to investigate, mutant pack is already feeding, conflict erupts). Both consequences subscribe to the same cause through the event bus. Neither references the other.

7. **Radiant consequence singularity.** A radiant cause evaluates one squad per tick. That squad is the actor. Only one consequence fires per evaluation. When multiple behaviors are possible for the same cause (stalker conquer vs mutant conquer vs mutant lair), they go into a config table as alternative entries with alignment, species, and smart filter gates. First match wins. Reactive causes have no such constraint because the event happens in the world and multiple squads respond independently.

8. **Domain gates in ext only.** Alignment and personality checks live exclusively in ext consequence code (`ap_ext_util`). Species filtering is a direct hash lookup in consequence code. Core pipeline has zero domain knowledge. Core handles rate limiting, protection, tracing, and squad lifecycle. Ext handles who acts and how likely.

### Consequence File Patterns

Three patterns for organizing consequences under a cause:

**CONFIGS (one file, alternatives).** Multiple behaviors for the same cause where only one fires per evaluation. A config table lists entries with alignment, species, personality, and smart filter gates. First match wins. Suited for radiant causes where the ticking squad picks one action. Examples: `stash` (loot/ambush/fill), `needs` (14 entries), `instincts` (4 entries).

**Separate files (cascading).** Multiple consequences that fire independently on the same cause event. Each subscribes to the cause via xbus and runs its own gate chain. Multiple can trigger from the same event. Suited for reactive causes where different actor types respond simultaneously. Examples: `massacre_investigate` + `massacre_scavenge`, `wounded_hunt` + `wounded_help`.

**_set file (grouped similar).** Multiple similar consequences in one file, each registered separately. Used when consequences share helper functions and chase/arrive logic but differ in alignment, species, or personality gates. The file is named `*_set.script`. Examples: `harvest_set` (harvest_rob + harvest_haunt).

---

## Integration API

AlifePlus exposes three levels of integration for external mods. See `integration-guide.md` for full examples and code templates.

![Integration Levels](img/integration-levels.png)

### Level 1: Listen (xbus subscriber)

Subscribe to cause events via xbus. No AP code dependency - xlibs only.

```lua
xbus.subscribe("cause:massacre", function(data)
    -- data.position, data.level_id, data.squad_id
end, "my_mod")
```

### Level 2: Register (pipeline participant)

Register a cause predicate with `ap_core_producer.register(name, config, predicate)` and a consequence handler with `ap_core_consumer.register(name, config, handler)`. The framework handles gates, protection, rate limiting, tracing, arrival, and cleanup.

| Parameter | Producer | Consumer |
|-----------|----------|----------|
| name | Cause identifier | Consequence identifier (also used as trace key, rate limit key, arrival key) |
| config.callback | Engine callback name | - |
| config.cause_type | RADIANT or REACTIVE | - |
| config.event | - | Cause event to subscribe to |
| config.on_arrive | - | Optional arrival handler function |
| handler | Predicate function | Consequence handler function |

### Level 3: Coordinate (external mod integration)

Register a squad ownership filter to prevent AP from scripting squads your mod controls.

```lua
ap_core_broker.register_owner("my_mod", function(squad)
    return squad.my_mod_flag == true
end)
```

Squads matching any registered ownership filter are excluded from AP at the protection gate (producer), in `find_squads` results (consequence), and via `get_owner` queries. Gated by MCM `allow_external_ownership`. Warfare and BAO are registered by default in `ap_core_compat`.
