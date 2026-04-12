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
| ap_core_consumer | Priority dispatch: xbus subscribe, consequence chain, result code propagation |
| ap_core_squad | Ownership registry, squad scripting lifecycle, arrival detection, protection, PDA markers |
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
| ap_ext_const | Community sets, faction lists, item pools |
| ap_ext_messages | PDA message templates |
| ap_ext_test | In-game debug commands |

---

## Core

### Event Pipeline

The producer receives engine callbacks and filters them through a chain of gates before evaluating cause predicates. Two variants exist because radiant and reactive events have fundamentally different characteristics.

**Radiant events** are ambient observations. A squad periodically scans its surroundings and notices something (a stash, empty territory, an unmet need). The squad is both sensor and responder. High frequency, low significance per event.

**Reactive events** are world-state changes. Something happened (a death, a healing, a pickup) and the framework reacts. The triggering entity may not be the entity AP acts on. Low frequency, high significance per event.

#### Radiant variant

Radiant causes fire on `squad_on_update` - the only stable, uniform heartbeat covering both online and offline squads. The engine fires this callback from `sim_squad_scripted:update()` for every squad every tick. Online squads fire at frame rate (~3/sec per squad), offline squads fire via the A-Life scheduler round-robin (~0.03/sec per squad, ~20 squads per tick across ~776 total). This produces a raw volume of ~6,600 calls/min that the gate chain must reduce to a manageable rate.

Radiant causes are an open collection. New causes can be added by registering a predicate on `RADIANT_CALLBACKS` - the pipeline evaluates all registered predicates on every admitted call, treating them as one pool. This is different from reactive causes, where each cause registers on a specific callback type.

The triggering squad is both sensor and responder - it evaluates its own surroundings and acts on what it finds (stashes, empty territory, unmet needs).

**Gate 1: PACER.** `os.clock` timestamp comparison. Rejects all calls within `cfg.distributor_interval_sec` of the last accepted call. Rejects ~99% of `squad_on_update` volume at near-zero cost (one number comparison, no luabind). On protection reject (gate 4), the pacer retries after a shorter interval instead of burning the full cooldown.

**Gate 2: SCRIPTED.** Hash lookup in `_ap_scripted_squads`. Skips squads already under AP control - a squad executing a consequence should not trigger new causes until released. O(1), no luabind.

**Gate 3: RATIO.** Bresenham integer admission gate. Off-map events outnumber on-map ~50-100:1 per squad because online squads fire at frame rate while offline squads fire via scheduler round-robin. The ratio gate restores balance using integer cross-multiplication: `throttled_count * |r| <= (10 - |r|) * favored_count`. At the default ratio of 8, this admits ~4 on-map events per 1 off-map. The `squad.online` field (C++ `m_bOnline`, refreshed by `check_online_status()` immediately before the callback) determines on-map status with zero luabind cost. Radiant and reactive pipelines maintain separate counter pairs (`_radiant_ct`, `_reactive_ct`) so high-volume radiant traffic does not bias reactive admission. Counters reset at 32768 to prevent overflow.

**Gate 4: PROTECTION.** Calls `xsquad.is_protected()` with the ownership registry. Rejects squads that are permanent (story NPCs, traders, named characters), have an active role (task giver, companion), are task targets (assault, bounty, delivery squads), are owned by another mod (warfare, BAO), or are already scripted (engine `scripted_target`, condlist, random_targets). Static checks are cached per squad with weak keys (O(1) after first call).

Protection is not a single gate - it is applied at four layers across the pipeline. The same guard set is checked at each layer, but against different entities depending on context:

| Layer | Where | What is checked |
|-------|-------|-----------------|
| Producer | Radiant gate 4/5 | Triggering squad, before any cause evaluates |
| Cause | Reactive predicates | The entity AP would act on (killer, patient, taker) |
| Consequence | `ap_core_utils.find_squads` | Every candidate responder squad |
| Squad | `ap_core_squad.script_squad` | No inline check - protection is upstream |

Reactive causes skip the producer protection gate because the callback entity (e.g. a dead victim in `squad_on_npc_death`) is not the entity AP would script. Instead, reactive causes check protection on the relevant entity inside the predicate itself: `alpha` and `alphakill` guard the killer, `wounded` guards the patient, `harvest` guards the taker. Causes where the trigger is a dead victim (`massacre`, `squadkill`, `basekill`) need no guard - consequences find responders through `find_squads` which applies all exclusions.

`script_squad` does not check protection. It assumes all upstream layers have already verified the squad. Direct callers outside the pipeline must check `is_protected` themselves.

**Gate 5: EVAL.** Iterates all registered radiant cause predicates. Each predicate is wrapped in `observe()` for tracing and checked against a per-cause sliding window rate limit (`cfg.cause_max_events`, 60s window). A mid-chain guard (`_scripted_during_eval`) prevents competing predicates from overriding each other's squad destinations - if handler A scripts the squad, handler B is skipped.

#### Reactive variant

Reactive causes fire on world events: `squad_on_npc_death`, `x_npc_medkit_use`, `actor_on_item_take`, `npc_on_item_take`, `actor_on_item_use`. These are low-frequency, high-significance events (a death, a healing, a pickup). The gate chain is shorter because the triggering entity may be a dead victim or an uninvolved bystander - protection must happen downstream in causes and consequences where the actual responder is known.

**Gate 1: PACER.** Token bucket with per-callback-type keys, 1 token/sec per type. Independent from the radiant pacer.

**Gate 2: RATIO.** Same Bresenham algorithm as radiant but with its own counter pair (`_reactive_ct`). Some reactive callbacks (`actor_on_item_use`, `actor_on_item_take`) are always on-map (they require a game_object which only exists online).

**Gate 3: EVAL.** Same as radiant gate 5 - per-cause rate limit, predicate evaluation, xbus publish.

### Dispatch Pipeline

After a cause publishes to xbus, the consumer routes the event to registered consequence handlers via round-robin. Each cause event type maintains a cursor that rotates which consequence gets tried first, ensuring equal distribution across all handlers in a chain.

**Global rate limit.** Sliding window TTL counter across radiant consequences only (`cfg.global_consequence_max_events`, 60s window). When the global budget is exhausted, the radiant consequence loop breaks. Reactive consequences (from deaths, wounds, pickups) bypass the global limit entirely -- they are rare, high-significance events that should never be starved by ambient radiant activity.

**Per-type rate limit.** Token bucket with peek-then-acquire. Each consequence type has its own budget (`cfg.consequence_max_events`, 60s window). Peek checks availability without consuming; acquire happens only after the handler succeeds.

**Handler execution.** Optional condition pre-filter (from registration). Then the handler function runs and returns a result code.

**Chain control.** Two stop codes: `OK_STOP` (success, exclusive - this consequence claimed the event) and `ERROR_STOP` (failure, abort chain). All other codes propagate: `OK_NEXT` (success, non-exclusive), `RULES_NEXT` (conditions unmet), `DISABLED_NEXT` (MCM off), `THROTTLED_NEXT` (rate limited). On success (`OK_STOP` or `OK_NEXT`), the per-type counter increments, and the global counter increments for radiant events only.

### Rate Limiting

All rate limiting lives in `ap_core_limiter`. Five independent limiters operate at different scopes.

| Limiter | Mechanism | Scope | Default | Config |
|---------|-----------|-------|---------|--------|
| Radiant pacer | os.clock timestamp | global | 10s | MCM distributor_interval_sec |
| Reactive pacer | token bucket | per-callback-type | 1/sec | constant |
| Cause budget | TTL counter, sliding window | per-cause | 10/60s | MCM cause_max_events |
| Consequence budget | token bucket (peek/acquire) | per-consequence | 2/60s | MCM consequence_max_events |
| Global consequence | TTL counter | radiant only | 6/60s | MCM global_consequence_max_events |

### Tracing

Hierarchical tracing via `observe()` in `ap_core_debug`. Each trace carries a monotonic **tid** (trace ID) and a slash-separated **path** (span hierarchy). A single `tid` links a cause through its consequence chain into individual actions:

```
[CAUSE.NEEDS] [tid=42 path=CAUSE.NEEDS] ok id=1337 need=hunger drive=4.2 [0.15ms]
[CONSEQUENCE.HUNGER_CAMPFIRE] [tid=42 path=CAUSE.NEEDS/CONSEQUENCE.HUNGER_CAMPFIRE] ok_stop count=1 [0.83ms]
[ACTION.FIND_DESTINATION] [tid=42 path=CAUSE.NEEDS/CONSEQUENCE.HUNGER_CAMPFIRE/ACTION.FIND_DESTINATION] ok id=445 [0.12ms]
```

Below DEBUG log level: `observe()` is a bare passthrough (calls the function, returns the result). `trace()` returns a null singleton. Cost: one `enabled()` check (~150ns) per call. The `_null_trace` and associated no-ops are pre-allocated singletons - no allocation at non-debug levels.

### Squad Lifecycle

`ap_core_squad` manages the full lifecycle of squads under AP control: scripting, arrival detection, post-arrival wait, and release.

**Scripting.** `script_squad(squad, smart, opts)` sets `scripted_target` via `xsquad.control_squad`, which causes the squad to enter `specific_update` (direct A->B movement) instead of `generic_update` (simulation re-evaluation). The squad is registered in `_ap_scripted_squads` with a TTL, optional arrival handler, and wait duration. `script_actor_target(squad)` scripts a squad to pursue the player using engine-native actor targeting (no arrival detection).

**Scripted squad scan.** `_update_scripted_squads` runs every 20s via `CreateTimeEvent`. For each tracked squad:

1. **TTL** - 7200 game-seconds. Expired -> unscript. Prevents permanently pinned squads.
2. **Arrival** - `xsmart.is_arrived(squad, smart)`. On arrival: if smart is full and squad is online, fire the arrival handler then unscript (overflow). Otherwise, dispatch the registered `on_arrive` function, then enter wait state.
3. **Wait** - `release_at = game_sec() + post_arrived_wait` (default 300s). When game time exceeds `release_at`, unscript. Game time advances during sleep/time-skip and survives save/load.

**Save/load.** `_ap_scripted_squads` persists to `m_data.ap_core_squad`. Engine-side `scripted_target` persists natively across save/load (`sim_squad_scripted` STATE_Write/STATE_Read). Arrival handler functions are transient - they are re-registered every load via `consumer.register` opts. On load, squads marked as arrived get `release_at = 0` (immediate release on next scan).

**Coordination.** `scripted_target` is the engine's built-in coordination mechanism between alife mods. Any mod that sets `scripted_target` signals "I control this squad." AP checks this at two gates: SCRIPTED (gate 2, AP's own squads) and PROTECTION (gate 4, anyone's squads). Two alife mods that both check `scripted_target` before claiming a squad will not conflict. The ownership registry (`register_owner`) adds identity on top: it tells AP WHO owns a squad, not just that it's owned. Warfare and BAO are registered by default in `ap_core_compat`.

**Markers.** `add_marked_squad(squad_id, label, opts)` adds a squad to the PDA map marker system. A 10s timer applies new marks; a 20s timer validates (removes dead or unscripted squads). The `persistent` flag keeps a marker until the entity dies regardless of scripted status (used by `alpha_promote`).

### Initialization Lifecycle

Four phases. Each requires the previous to complete.

**Phase 0: Module load.** The engine's auto-load mechanism (`_G.__index = auto_load` in `script_engine.cpp:375`) resolves `.script` files on first namespace access. `axr_main.on_game_start()` iterates all `.script` files in alphabetical order, triggering auto-load for each. Module-level code runs immediately on load: engine globals are cached to locals, constant tables are built, xlibs API references are captured. No game state exists at this point - no actor, no entities, no callbacks active.

**Phase 1: on_game_start.** `axr_main` calls `on_game_start()` on every loaded script. File order is alphabetical, but all operations are independent - no cross-module reads at this phase:

- `_ap_deps` asserts xlibs compatibility. Hard crash on mismatch.
- `ap_core_mcm` loads config from defaults, registers `on_option_change`.
- `ap_core_debug` registers `actor_on_first_update` for deferred log level init.
- `ap_core_producer` resets dispatch state, registers `actor_on_first_update`. Does NOT subscribe to callbacks yet.
- `ap_core_consumer` registers `actor_on_first_update`. Does NOT subscribe to xbus yet.
- `ap_core_squad` registers save/load/first_update callbacks, creates the 20s scripted squad scan timer.
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

Domain state manager. Tracks kill counts per entity (`_ap_killers`), alpha status with level/kills/name (`_ap_alphas`), alpha death grace period (`_ap_alpha_dead`, xttltable TTL, 3600s), and stalker needs DTO (`_ap_stalker_needs`, per-squad timestamps for 9 drives). Only mutants become alphas -- stalker rank is handled natively by the engine. Registers the `squad_on_npc_death` handler for kill/alpha tracking. Save/load to `m_data.ap_ext_tracker`.

### Smart Mutator (ap_ext_smart_mutator)

Runtime smart terrain mutations for territory conquest. Both stalker and mutant factions can conquer smart terrains.

**Mechanism.** `conquer_smart(smart_id, faction)` calls `xsmart.set_faction_controlled` which writes `respawn_params` entries for all 19 factions (12 stalker + 7 mutant) and sets `smart.faction` to the conqueror. The engine's `try_respawn` (smart_terrain.script:1665) only spawns entries where `v.faction == self.faction`, so only the conqueror's squads appear. Stalker factions spawn `*_sim_squad_novice/advanced/veteran` sections. Mutant factions spawn `simulation_*` sections mapped by species (e.g. `monster_predatory_day` spawns dogs and pseudodogs).

**Volatility.** The engine rebuilds smart config from LTX on every load (`STATE_Read`), so the mutator uses two-phase restore (`load_state` reads to pending, `on_game_load` re-applies).

**Monster faction revert.** `check_smart_faction` (smart_terrain.script:1209) only counts `IsStalker` NPCs. When only monsters occupy an online smart, `smart.faction` reverts to `default_faction`. A 60s periodic scanner (`_scan_conquered`) re-applies `smart.faction` for monster-conquered smarts.

**Decay.** Conquered smarts revert to their original faction after `cfg.area_conquest_decay_hours` game hours (default 48). The scanner checks `xtime.game_sec() - conquered_at` and calls `clear_faction_controlled` on expired entries. Decay creates a natural territorial cycle: factions conquer, territory decays, other factions reconquer.

**Eviction.** FIFO at `cfg.area_conquest_max_smarts` (default 50). Oldest conquest by game time is evicted when the cap is reached. Same-faction re-conquest refreshes the timestamp (LRU). Different-faction overwrites without eviction.

### Object Mutator (ap_ext_object_mutator)

Runtime combat modifiers for alpha mutants and high-rank stalkers. Two independent systems on `monster_on_before_hit` and `npc_on_before_hit`.

**Alpha mutants** (`monster_on_before_hit`): outgoing hit power bonus and incoming hit power reduction via `_alpha_damage_increase[npc_id]` / `_alpha_damage_reduction[npc_id]` hash tables, populated at promote time. Panic immunity (`set_custom_panic_threshold(0)`) applied lazily on first hit. O(1) lookup, 0.5s throttle, early exit when tables empty. Alpha level: `min(10, floor(kills / alpha_kills_per_level))`. Loot items injected at promote time via `xobject.create_item` from tiered pools (LOW/MID/HIGH), managed in `alpha_promote` consequence.

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

Predicate contract: `function(trace, ...callback_args) -> { cause = CAUSE.X, ...payload } | nil`. Producer wraps each call in `observe()`, attaches `._trace`, publishes to xbus, increments cause counter. Predicates only evaluate and return - no observe(), publish(), trace creation, or counter manipulation.

### Consequences

| Cause | Consequence | Effect |
|-------|-------------|--------|
| MASSACRE | massacre_scavenge | scavenger factions converge on massacre site |
| MASSACRE | massacre_investigate | victim faction investigates |
| SQUADKILL | squadkill_revenge | victim faction pursues killer (chase) |
| SQUADKILL | squadkill_flee | victim faction retreats to base |
| BASEKILL | basekill_support | friendly squads reinforce attacked base |
| BASEKILL | basekill_flee | squads at base evacuate |
| ALPHA | alpha_promote | update alpha level, apply hit power/panic buffs, grant loot |
| ALPHAKILL | alphakill_targeted | other alphas pursue killer (chase) |
| WOUNDED | wounded_hunt | predator mutants close in |
| WOUNDED | wounded_help | same-faction squads rush to help |
| HARVEST | harvest_hunt | outlaws pursue artefact taker (chase) |
| STASH | stash_loot | loot stash to NPC inventory (CONFIGS-driven) |
| STASH | stash_ambush | camp at stash, passive (CONFIGS-driven) |
| STASH | stash_fill | fill stash with items (CONFIGS-driven) |
| AREA | area_conquer | claim empty smart terrain (stalkers and mutants, decays after 48h) |
| NEEDS | (14 entries) | hunger, sleep, rest, heal, shelter, money, supply, job, social (CONFIGS-driven) |

Handler contract: `function(event_data) -> { code = RESULT.X, reason = "..." }`. Gate order inside each handler: enabled check -> alignment check -> data validation -> logic -> result code. Dispatch order: round-robin cursor per cause type, personality chance evaluated by consumer before handler runs.

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

Mutant factions follow GSC's creature groups (monstry.doc:4):

| Table | Factions | GSC origin |
|-------|----------|------------|
| alignment_predator | monster_predatory_day, monster_predatory_night | Roaming pack hunters |
| alignment_territorial | monster_predatory_night, monster_special | Hold ground, defend territory |

Consequences compose tables with `xtable.merge` (set union) and `xtable.subtract` (set difference) at module load.

For radiant consequences (stash, area, needs), the alignment check is on `event_data.community` -- the triggering squad's faction. For reactive same-faction consequences (investigate, revenge, flee, support, help), the check is on the event faction (victim or wounded). For reactive cross-faction consequences (scavenge, hunt), the alignment table is the `factions` parameter to `find_squads`.

### Personality

Probability layer: how likely is an eligible faction to act. Runs only after alignment passes. Seven traits (aggression, greed, survival, perception, territory, relation, discipline) declared per consequence at registration. The consumer averages the relevant traits for the event faction, clamps to 0.05-0.95, and rolls.

### Emergence

Per-level perceptron that observes consequence outcomes and adjusts consequence probability in real-time. Each map develops its own behavioral profile. Reads AP saturation, consequence throughput, and faction disposition. Weights adapt every session. See `ap_core_consumer` for integration.

### Invariants

1. **Serialization boundary.** `m_data` accepts only primitives and tables of primitives. Functions, userdata, metatables are silently dropped on save. Arrival handler keys are strings; handler functions are transient, re-registered every load.

2. **Result code contract.** Every consequence returns `{ code = RESULT.X }`. Consumer checks `.code` for chain propagation. Missing or malformed return = error.

3. **Scripted bypass.** `scripted_target` overrides `SIMBOARD:get_squad_target()`. Engine `target_precondition` (faction check, capacity check) is NOT evaluated for scripted squads. AP enforces faction safety in consequence filters. Capacity is handled by arrival overflow.

4. **Mutation volatility.** Runtime smart terrain mutations are rebuilt from LTX on every load. The smart mutator saves and restores independently using two-phase restore. A 60s periodic scanner re-applies monster faction ownership (engine's `check_smart_faction` ignores monsters) and decays expired conquests.

5. **Core/ext boundary.** Core never imports ext. All domain logic reaches core through registered function references.

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
ap_core_squad.register_owner("my_mod", function(squad)
    return squad.my_mod_flag == true
end)
```

Squads matching any registered ownership filter are excluded from AP at the protection gate (producer), in `find_squads` results (consequence), and via `get_owner` queries. Gated by MCM `allow_external_ownership`. Warfare and BAO are registered by default in `ap_core_compat`.
