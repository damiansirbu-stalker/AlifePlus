# AlifePlus Architecture

Reactive behavior framework for STALKER Anomaly. Engine callbacks produce causes, causes dispatch consequences through a gated pipeline. Built on xlibs.

See `conventions.md` for naming rules, result codes, MCM settings, logging format.

---

## Glossary

| Term | Definition |
|------|------------|
| Cause | Engine callback + world-state predicate -> meaningful event or nil |
| Consequence | Handler subscribed to a cause; executes response logic + side effects |
| Predicate | Pure function: `(trace, ...args) -> { cause, ...payload }` or nil |
| Producer | Dispatches callbacks through gate chain, evaluates predicates, publishes to xbus |
| Consumer | Receives cause events from xbus, dispatches to consequences by priority |
| xbus | Pub/sub event bus (xlibs). Causes publish, consequences subscribe |
| Core | Framework modules (ap_core_*). Pipeline, lifecycle, protection, rate limiting |
| Ext | Domain modules (ap_ext_*). Causes, consequences, data, messages |

---

## Core / Ext Split

AlifePlus separates framework from domain logic.

**Core** (10 files, ap_core_* + _ap_deps) -- the pipeline, protection, squad lifecycle, rate limiting, tracing, configuration. No game domain knowledge. Any cause/consequence mod plugs into core.

| File | Responsibility |
|------|---------------|
| ap_core_const | Enums: CAUSE, CONSEQUENCE, RESULT, REASON, CALLBACK, CAUSE_TYPE, TRACE, ACTION |
| ap_core_mcm | MCM defaults, config snapshot (cfg table), UI builder |
| ap_core_debug | Logger, observe() tracing, result helpers. Zero-cost below DEBUG |
| ap_core_producer | Gate chain, predicate evaluation, xbus publish |
| ap_core_consumer | Priority dispatch: xbus subscribe, consequence chain, result code propagation |
| ap_core_limiter | Rate limiting: cause counter, consequence token bucket, global consequence counter, cooldowns |
| ap_core_squad | Ownership registry, squad scripting lifecycle, arrival detection, protection, PDA markers |
| ap_core_utils | Event pub/sub wrappers, PDA dispatch, find_squads/find_smart with protection |
| ap_core_compat | Save data migrations, proxy ownership registrations (warfare, bao) |
| _ap_deps | Dependency gate: assert xlibs installed and version-compatible |

**Ext** (33 files, ap_ext_*) -- causes, consequences, tracker, conquest, data, messages, test tools. Domain logic that registers with core.

| File | Responsibility |
|------|---------------|
| ap_ext_const | Community sets, faction lists, item pools, section lists |
| ap_ext_tracker | Domain state: killers, elites, elite_dead, stalker_needs. Death handler |
| ap_ext_smart_mutator | Territory conquest: conquered_smarts, faction_controlled mutation, FIFO eviction |
| ap_ext_object_mutator | Runtime combat modifiers: grenades, AP ammo, rank boost (npc_on_before_hit) |
| ap_ext_messages | PDA message templates |
| ap_ext_test | In-game debug commands |
| ap_ext_cause_* | Cause predicates (one file per cause group) |
| ap_ext_consequence_* | Consequence handlers (one or more files per cause) |

**Boundary rule:** core never imports ext. Ext registers with core via `ap_core_producer.register()` and `ap_core_consumer.register()`. Core calls ext only through registered function references.

---

## Lifecycle

Four phases. Each requires the previous to complete.

### Phase 0 -- Module load

Lua require resolves all script files. Module-level code runs: engine globals cached to locals, constant tables built, xlibs API references captured. No game state exists.

### Phase 1 -- on_game_start

Engine calls on_game_start() per script. File order is alphabetical, operations independent:

- **_ap_deps** asserts xlibs compatibility. Hard crash on mismatch.
- **ap_core_mcm** loads config from defaults, registers on_option_change.
- **ap_core_debug** registers actor_on_first_update for deferred log level init.
- **ap_core_producer** resets dispatch state, registers actor_on_first_update. Does NOT subscribe to callbacks yet.
- **ap_core_consumer** registers actor_on_first_update. Does NOT subscribe to xbus yet.
- **ap_core_squad** registers save/load/first_update callbacks, creates scripted squad scan timer (10s).
- **ap_core_compat** registers load_state for migrations, registers ownership proxies (warfare, bao).
- **ap_ext_cause_*** register predicates with producer via `register()`.
- **ap_ext_consequence_*** register handlers with consumer via `register()`. Arrival handlers registered via consumer opts.

After: predicates/handlers registered, no callbacks or subscriptions active.

### Phase 2 -- actor_on_first_update

Game world + actor exist. Deferred init runs:

- **Producer** rebuilds radiant handler set from registrations, subscribes to engine callbacks.
- **Consumer** subscribes to xbus cause events.
- **Debug** reads MCM log level, dumps all settings.
- **Squad** clears stale markers, starts marker timer if MCM enabled.

### Phase 3 -- on_game_load

After STATE_Read rebuilds all game objects from save. Smart terrain mutations lost during STATE_Read. Conquest mutator re-applies conquered smarts from pending data (two-phase restore).

---

## Pipeline

Two entry points, shared evaluation. Producer dispatches, consumer routes.

### Radiant pipeline

Fires on `squad_on_update` (sole radiant source). The triggering squad is both sensor and responder.

```
squad_on_update callback
  |
  v
PACER (1/5)        os.clock timestamp, cfg.distributor_interval_sec (default 5s)
  |                 rejects ~99% of squad_on_update calls
  v
SCRIPTED (2/5)     hash lookup in _ap_scripted_squads
  |                 skip squads already under AP control
  v
RATIO (3/5)        Bresenham integer admission, on-map vs off-map
  |                 cfg.alife_ratio (-10..+10, default 8) -> ~4:1 on:off
  v
PROTECTION (4/5)   xsquad.is_protected + ownership registry
  |                 permanent, active_role, task_target, owned, scripted
  v
EVAL (5/5)         iterate all radiant cause predicates
  |                 per-cause rate limit (sliding window)
  |                 _scripted_during_eval mid-chain guard
  |                 predicate -> result -> xbus.publish
```

On protection reject: PACER retries after 0.5s instead of burning full interval.

### Reactive pipeline

Fires on world events: death, medkit use, item pickup.

```
callback (squad_on_npc_death, x_npc_medkit_use, actor_on_item_take, ...)
  |
  v
PACER (1/3)        token bucket, per-callback-type key (1/sec)
  |
  v
RATIO (2/3)        same Bresenham gate as radiant
  |
  v
EVAL (3/3)         iterate handlers for this callback
                    per-cause rate limit, predicate -> xbus.publish
```

No SCRIPTED or PROTECTION gate -- reactive callbacks may fire for dead victims or uninvolved entities. Protection happens downstream in consequences.

### Consumer dispatch

```
xbus cause event
  |
  v
GLOBAL RATE LIMIT   sliding window TTL counter (cfg.global_consequence_max_events, 60s)
  |
  v
PER-TYPE RATE LIMIT  token bucket peek (cfg.consequence_max_events, 60s)
  |
  v
CONDITION            optional pre-filter from registration
  |
  v
HANDLER              consequence function -> result code
  |
  v
CHAIN CONTROL        OK_STOP/ERROR_STOP breaks, _NEXT continues
```

Consequences execute in priority order (lower number = first). On success (OK_STOP or OK_NEXT), global and per-type counters increment.

---

## Registration API

### Registering a cause

```lua
ap_core_producer.register(name, config, predicate)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| name | string | Cause identifier (e.g. CAUSE.MASSACRE) |
| config.callback | string | Engine callback name (e.g. CALLBACK.SQUAD_ON_NPC_DEATH) |
| config.cause_type | string | CAUSE_TYPE.RADIANT or CAUSE_TYPE.REACTIVE (default: REACTIVE) |
| predicate | function | `(trace, ...callback_args) -> { cause, ...payload } or nil` |

Radiant causes register once per RADIANT_CALLBACKS entry (currently just CALLBACK.SQUAD_ON_UPDATE). Producer collapses all radiant registrations into one flat handler table.

### Registering a consequence

```lua
ap_core_consumer.register(name, config, handler)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| name | string | Consequence identifier (e.g. CONSEQUENCE.MASSACRE_SCAVENGE) |
| config.event | string | Cause event to subscribe to (e.g. CAUSE.MASSACRE) |
| config.priority | number | Execution order (lower = first, default 50) |
| config.condition | function | Optional pre-filter: `() -> boolean` |
| config.on_arrive | function | Optional arrival handler: `(squad, args)` |
| handler | function | `(event_data) -> { code = RESULT.X, reason = "..." }` |

The consequence name is used as the trace key, rate limit key, and arrival handler key. One identity for all three.

---

## Protection

Four guard types checked via `xsquad.is_protected()`, plus an ownership registry.

| # | Guard | Nature | What it catches |
|---|-------|--------|-----------------|
| 1 | is_permanent_squad | static (cached) | story_id, trader, named_npc, empty_squad |
| 2 | has_active_role | dynamic | task_giver, companion |
| 3 | is_task_target | dynamic | task system squads (assault, bounty, delivery, etc.) |
| 4 | is_scripted | dynamic | scripted_target, action_condlist, random_targets |

### Ownership registry

Mods register ownership filters via `ap_core_squad.register_owner(name, filter_fn)`. Squads matching any registered filter are treated as owned and excluded from AP operations.

```lua
-- ap_core_compat registers these at on_game_start:
ap_core_squad.register_owner("warfare", function(squad)
    return squad.registered_with_warfare == true
end)
ap_core_squad.register_owner("bao", function(squad)
    return squad.__lock == true
end)
```

`get_owner(squad)` returns the owner name or nil. Gated by MCM `allow_external_ownership`.

### Protection layers

Protection is applied at four layers:

| Layer | Where | What |
|-------|-------|------|
| Producer | radiant gate 4/5 | `is_protected` on triggering squad before any cause evaluates |
| Cause | reactive predicates | `is_protected` on the entity AP would act on (killer, patient, taker) |
| Consequence | `ap_core_utils.find_squads` | all four guard types + ownership as exclusion filters |
| Squad | `ap_core_squad.script_squad` | no inline protection (removed; upstream gates + find_squads handle it) |

---

## Squad Lifecycle (ap_core_squad)

### Scripting

```
script_squad(squad, smart, opts)
  -> release if already scripted
  -> xsquad.control_squad (sets scripted_target, rush_to_target)
  -> register in _ap_scripted_squads with TTL, arrival handler, wait duration

script_actor_target(squad)
  -> register as "actor" target (engine-native pursuit, no arrival detection)
```

`scripted_target` bypasses SIMBOARD:get_squad_target() -- squad enters direct A->B movement instead of simulation re-evaluation. Persists across save/load natively.

### Housekeeping scan

`_update_scripted_squads` runs every 10s via CreateTimeEvent. For each tracked squad:

1. **TTL check** -- 7200 game-seconds (SCRIPTED_SQUAD_TTL). Expired -> unscript.
2. **Arrival check** -- `xsmart.is_arrived(squad, smart)`. On arrival:
   - Overflow: if smart is full and squad is online, fire handler then unscript.
   - Handler: dispatch registered on_arrive function.
   - Wait: set `release_at = game_sec() + post_arrived_wait` (default 300s).
3. **Wait check** -- release squad when game time exceeds release_at.

### Save/load

`_ap_scripted_squads` persisted to `m_data.ap_core_squad`. On load, squads marked as arrived get `release_at = 0` (immediate release on next scan). Handler functions are transient -- re-registered every load via consumer.register opts.

---

## Rate Limiting (ap_core_limiter)

| Limiter | Type | Scope | Window | Config |
|---------|------|-------|--------|--------|
| Radiant pacer | os.clock timestamp | global | cfg.distributor_interval_sec (default 5s) | MCM 1-30s |
| Reactive pacer | token bucket | per-callback-type | 1/sec (REACTIVE_EVENT_INTERVAL_SEC) | constant |
| Cause budget | TTL counter | per-cause | 60s sliding window | cfg.cause_max_events (MCM) |
| Consequence budget | token bucket (peek then acquire) | per-consequence | 60s window | cfg.consequence_max_events (MCM) |
| Global consequence | TTL counter | global | 60s window | cfg.global_consequence_max_events (MCM) |

**Bresenham ratio gate.** Off-map events outnumber on-map ~50:1. Integer cross-multiplication for on:off admission ratio. At ratio=8: ~4:1 on:off. Formula: `throttled_count * |r| <= (10 - |r|) * favored_count`. Reset at 32768.

---

## Tracing (ap_core_debug)

Hierarchical tracing via `observe()`. Each trace carries:

- **tid** -- monotonic trace ID. Grep `tid=N` in alifeplus.log to follow a cause->consequence chain.
- **path** -- slash-separated span hierarchy (e.g. `CAUSE.NEEDS/ACTION.FIND_DESTINATION/ACTION.MOVE_SQUAD`).

Log format: `[OP_LABEL] [tid=N path=A/B] code:reason key=val [Xms]`

Below DEBUG: observe() is a bare passthrough. trace() returns a null singleton. Cost: one `enabled()` check per call.

---

## Result Codes

OUTCOME_PROPAGATION naming. Name encodes outcome + chain behavior.

| Code | Outcome | Chain |
|------|---------|-------|
| OK_STOP | success, exclusive | stops |
| OK_NEXT | success, non-exclusive | continues |
| RULES_NEXT | conditions unmet | continues |
| CHANCE_NEXT | roll failed | continues |
| DISABLED_NEXT | MCM off | continues |
| THROTTLED_NEXT | rate limited | continues |
| ERROR_STOP | failure | stops |

Two stoppers only: OK_STOP and ERROR_STOP.

---

## Causes

| Cause | Type | Callback(s) | Conditions |
|-------|------|-------------|------------|
| MASSACRE | reactive | squad_on_npc_death | IsStalker(victim), at smart, not is_base, kill_count >= threshold |
| SQUADKILL | reactive | squad_on_npc_death | last member dead, not is_base |
| BASEKILL | reactive | squad_on_npc_death | IsStalker(victim), at is_base, kill_count >= threshold |
| ELITE | reactive | squad_on_npc_death | killer not protected, projected kills cross new level |
| ELITEKILL | reactive | squad_on_npc_death | victim is elite, killer not protected, cooldown clear |
| WOUNDED | reactive | x_npc_medkit_use, actor_on_item_use | subject not protected, not is_base |
| HARVEST | reactive | actor_on_item_take, npc_on_item_take | IsArtefact(item), NPC taker not protected |
| STASH | radiant | squad_on_update | not protected, stash within RADIUS_DISTANT_MAX |
| AREA | radiant | squad_on_update | not protected, community_stalker, empty smart, not is_base |
| NEEDS | radiant | squad_on_update | not protected, community_stalker, level.present(), Hull drive scoring |

### Predicate contract

Signature: `function(trace, ...callback_args) -> { cause = CAUSE.X, ...payload } | nil`

Producer wraps each call in observe(), attaches `._trace`, publishes to xbus, increments cause counter. Predicates only evaluate and return -- no observe(), publish(), trace creation, or counter manipulation.

---

## Consequence Patterns

Gate sequence per consequence: enabled -> chance -> logic -> result code. Rate limiting handled by consumer (upstream).

| Pattern | Movement | Examples |
|---------|----------|---------|
| respond | find nearby squads, script toward event location | massacre_scavenge, massacre_investigate, basekill_support, wounded_hunt, wounded_help |
| chase | arrival-based pursuit loop, re-script on each arrival | squadkill_revenge, elitekill_targeted, harvest_hunt |
| flee | script to nearest friendly base | squadkill_flee, basekill_flee |
| state | no movement, update tracker/rewards | elite_promote, elitekill_bounty |
| stash | script to stash's nearest smart, arrival handler for inventory ops | stash_loot, stash_fill, stash_ambush |
| conquest | script to empty smart, conquer on arrival | area_conquer |
| needs | config-driven (16 entries), triggering squad IS the responder | hunger_campfire, sleep_campfire, ... |

**Chase.** Each consequence inlines its own chase logic (no shared chase API). On arrival: alive? same level? not co-located? under max_chases? Pass -> re-script. Exceed max -> unscript. Player targets use script_actor_target.

**Needs.** Single predicate, 16 consequence configs. Hull drive scoring: `drive = weight * (elapsed / threshold)^2`. Maslow-weighted (SHELTER 5.0 to SOCIAL 0.8). Highest drive wins. NPCs consume real inventory items on arrival.

**Markers.** Each consequence calls `add_marked_squad(squad_id, label, opts)` after successful script_squad. Squad module's 5s timer applies marks; 60s timer validates (removes dead/unscripted). Persistent flag keeps marker until entity death (used by elite_promote).

---

## Consequences

| Cause | Consequence | p | Pattern | Effect |
|-------|-------------|---|---------|--------|
| MASSACRE | massacre_scavenge | 10 | respond | scavenger factions converge on massacre site |
| MASSACRE | massacre_investigate | 20 | respond | victim faction investigates |
| SQUADKILL | squadkill_revenge | 15 | chase | victim faction pursues killer |
| SQUADKILL | squadkill_flee | 25 | flee | victim faction retreats to base |
| BASEKILL | basekill_support | 15 | respond | friendly squads reinforce attacked base |
| BASEKILL | basekill_flee | 25 | flee | squads at base evacuate |
| ELITE | elite_promote | 15 | state | update elite level, apply buffs |
| ELITEKILL | elitekill_bounty | 0 | state | give_money (base * level, capped) |
| ELITEKILL | elitekill_targeted | 10 | chase | elite hunters pursue killer |
| WOUNDED | wounded_hunt | 15 | respond | predator mutants close in |
| WOUNDED | wounded_help | 25 | respond | same-faction squads rush to help |
| HARVEST | harvest_hunt | 15 | chase | outlaws pursue artefact taker |
| STASH | stash_loot | 10 | stash | loot stash to NPC inventory |
| STASH | stash_ambush | 20 | stash | camp at stash (passive) |
| STASH | stash_fill | 30 | stash | fill stash with items |
| AREA | area_conquer | 20 | conquest | claim empty smart terrain |
| NEEDS | (16 entries) | 10-40 | needs | hunger, sleep, rest, heal, shelter, money, supply, job, social |

---

## Domain Systems (Ext)

### Tracker (ap_ext_tracker)

Domain state manager. Killers, elites, elite_dead (TTL grace), stalker_needs (per-squad DTO). Save/load to `m_data.ap_ext_tracker`. Death handler on squad_on_npc_death.

### Smart Mutator (ap_ext_smart_mutator)

Runtime smart terrain mutations. Currently: territory conquest (sets `faction_controlled` and `respawn_params` so engine `try_respawn` spawns conqueror's faction). FIFO eviction at `area_conquest_max_smarts` (MCM, default 50). Two-phase save/load (mutations lost at STATE_Read, re-applied at on_game_load). Will contain all future smart terrain mutations.

### Object Mutator (ap_ext_object_mutator)

Runtime combat buffs from tracker kill data, on npc_on_before_hit. Elite level: `min(10, floor(kills / elite_kills_per_level))`. Grenades (count = level), AP ammo (best k_ap for weapon, boxes = ceil(level/2)), rank boost (threshold table). Gated by acquire_lock (1s).

---

## Invariants

1. **Serialization boundary.** m_data accepts only primitives and tables of primitives. Functions, userdata, metatables silently dropped on save. Arrival handler keys are strings; handler functions are transient, re-registered every load.

2. **Result code contract.** Every consequence returns `{ code = RESULT.X }`. Consumer checks .code for chain propagation. Missing or malformed return = error.

3. **Scripted bypass.** scripted_target overrides SIMBOARD:get_squad_target(). Engine target_precondition is NOT evaluated for scripted squads. AP enforces faction safety in consequence filters. Capacity handled by arrival overflow.

4. **Mutation volatility.** Runtime smart terrain mutations rebuilt from LTX on every load. Conquest mutator saves/restores independently (two-phase).

5. **Core/ext boundary.** Core never imports ext. All domain logic reaches core through registered function references (predicates, handlers, arrival callbacks).

---

## Extension

**New cause:** add CAUSE.X to ap_core_const, create ap_ext_cause_x.script with predicate, register via `ap_core_producer.register()`. Radiant: iterate RADIANT_CALLBACKS.

**New consequence:** add CONSEQUENCE.X to ap_core_const, create handler returning result codes, register via `ap_core_consumer.register()` with event, priority, optional on_arrive. Gate order: enabled -> chance -> logic. OK_STOP = exclusive, _NEXT = propagate.

**New ownership filter:** call `ap_core_squad.register_owner(name, filter_fn)` in on_game_start.

**External subscriber:** call `xbus.subscribe(cause_event, handler, name)` in on_game_start. No AP dependency needed beyond xlibs.
