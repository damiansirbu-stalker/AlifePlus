# AlifePlus Architecture

Behavior mod for STALKER Anomaly. Classifies A-Life events into causes, dispatches consequences. Built on xlibs.

See `conventions.md` for naming rules, result codes, MCM settings, logging format.

---

## Glossary

| Term | Definition |
|------|------------|
| Cause | Engine callback + world-state predicate -> meaningful situation or nil |
| Consequence | Handler subscribed to a cause event; executes business logic + side effects |
| Predicate | Pure function: `(trace, ...args) -> { cause, ...payload }` or nil |
| Producer | Dispatches X-Ray callbacks to registered predicates, publishes results to xbus |
| Consumer | Receives cause events from xbus, dispatches to consequences by priority |
| xbus | Pub/sub event bus (xlibs). Causes publish, consequences subscribe |

---

## System Context

No engine patches, no disk writes. Runtime callbacks, Lua globals, xlibs wrappers.

| System | What AP uses | Interface |
|--------|-------------|-----------|
| X-Ray callbacks | squad_on_npc_death, enter/leave_smart, actor_on_interaction, npc_on_before_hit, actor_on_item_use/take, npc_on_item_take, save/load_state, on_game_load, actor_on_first_update | RegisterScriptCallback |
| X-Ray A-Life | alife(), alife_object, alife_release_id, alife_create_item | Engine globals |
| X-Ray game state | db.storage, db.actor, level.present(), level.get_start_time(), level.get_time_hours(), game.get_game_time() | Engine globals |
| Anomaly simulation | scripted_target, rush_to_target | sim_squad_scripted (via xsquad) |
| Anomaly smart_terrain | faction_controlled, respawn_params, default_faction, try_respawn | Runtime table mutation (via xsmart) |
| Anomaly scripts | xr_reach_task (run+danger), xr_eat_medkit (NPC medkit hook) | xevent.hook, direct reference |
| Save system | m_data table in save_state/load_state callbacks | Anomaly serialization |
| xlibs | xbus, xsmart, xsquad, xcreature, xlevel, xlog, xpda, xmath, xtable, xttltable, xevent, xmcm, xinspect, xtime, xobject, xconst, xstash | Lua require (version-gated by _ap_deps) |

---

## Files

| File | Responsibility |
|------|---------------|
| _ap_deps | Dependency gate: assert xlibs installed and version-compatible |
| ap_const | Enums: CAUSE, CONSEQUENCE, RESULT, REASON, ACTION, CALLBACK, NEEDS_SECTIONS |
| ap_data | Community sets, faction lists, grenades, item data |
| ap_mcm | MCM defaults, getter, loader, UI builder |
| ap_debug | Logger, observe() tracing, result helpers |
| ap_producer_reactive | Unified producer: callback dispatch, gates, predicate eval, xbus publish |
| ap_consumer | Priority dispatch: xbus subscribe, consequence chain, rate limit |
| ap_alife_tracker | State manager: killers, elites, scripted_squads, stalker_needs |
| ap_smart_mutator | Territory conquest: conquered_smarts, faction_controlled mutation, FIFO eviction |
| ap_object_mutator | Runtime combat modifiers: grenades, AP ammo, rank boost (npc_on_before_hit) |
| ap_utils | Event wrappers, PDA, capacity-aware find_smart, chase API, find helpers |
| ap_cause_* | Cause predicates (one file per cause group) |
| ap_consequence_* | Consequence handlers (one file per cause; needs = config table) |
| ap_messages | PDA message templates |
| ap_test | In-game debug commands: force needs, spawn squads, trigger causes |

---

## Lifecycle

Four phases. Each requires the previous to complete.

### Phase 0 -Module load

Lua require resolves all script files. Module-level code runs: engine globals cached to locals, constant tables built, xlibs API references captured. No game state exists yet.

### Phase 1 -on_game_start

Engine calls on_game_start() per script. File order undefined; operations independent:

- **_ap_deps** asserts xlibs is_compatible(1.2.2). Hard crash on mismatch.
- **ap_mcm** loads config from defaults, registers on_option_change.
- **ap_alife_tracker** creates elite_dead TTL table, registers save_state/load_state/on_game_load/squad_on_npc_death, starts arrival timer (30s) and optional periodic_sync.
- **ap_object_mutator** registers npc_on_before_hit.
- **ap_producer_reactive** resets dispatch state, registers actor_on_first_update. Does NOT subscribe to callbacks yet.
- **ap_consumer** registers actor_on_first_update. Does NOT subscribe to xbus yet.
- **ap_cause_*** register predicates with producer via register().
- **ap_consequence_*** register handlers with consumer via register(). Register arrival handlers with tracker where needed.

After: predicates/handlers registered, no callbacks/subscriptions active.

### Phase 2 -actor_on_first_update

Game world + actor exist.

- Producer subscribes to engine callbacks + actor_on_interaction radiant adapter.
- Consumer subscribes to xbus cause events.
- Debug dumps MCM settings, utils clears stale squad markers.

### Phase 3 -on_game_load

After STATE_Read rebuilds all game objects from save. Smart terrain mutations lost during STATE_Read. Tracker re-applies conquered smarts from _conquered_pending (two-phase restore).

---

## Pipeline

```
ENGINE CALLBACKS                 xevent HOOKS
  squad_on_npc_death               xr_eat_medkit hook
  squad_on_enter_smart               -> x_npc_medkit_use
  squad_on_leave_smart
  actor_on_interaction (adapter)
         |                                |
         +----------------+---------------+
                          |
                          v
         PRODUCER (ap_producer_reactive)
           1. Level classification
           2. Bresenham ratio gate
           3. Event pacer (split radiant/reactive)
           4. Cause budget (sliding window)
           5. Predicate evaluation -> xbus.publish
                          |
                          v
         CONSUMER (ap_consumer)
           1. Consequence budget (token bucket)
           2. Condition pre-filter
           3. Handler execution -> result code
           4. Chain control (OK_STOP/ERROR_STOP breaks)
                          |
              +-----------+-----------+
              v           v           v
         [scavenge]  [investigate]  [revenge]
            p=10        p=20         p=15
```

---

## Tracing

Hierarchical tracing via observe() (ap_debug). Each trace carries:

- **tid** -monotonic trace ID. Grep `tid=N` in alifeplus.log to follow a cause->consequence chain.
- **path** -slash-separated span hierarchy (e.g. `CAUSE.NEEDS/ACTION.FIND_DESTINATION/ACTION.MOVE_SQUAD`). observe() pushes segment, times fn, logs result + path, pops.

Log format: `[OP_LABEL] [tid=N path=A/B] code:reason key=val [Xms]`

Producer creates root trace. Consumer passes via event_data._trace. Handlers extend via observe().

Below DEBUG: straight pass-through. _null_trace/_null_timer are no-op singletons. Cost: one lookup + one no-op per call.

---

## Design Principles

- **YANS (You Are Not Special).** Same rules for player and NPCs.
- **Low idle cost.** Event-driven. Two timers total: arrival check (30s) and periodic cleanup.
- **Decoupled.** Causes publish to xbus, consequences subscribe. Neither references the other. Engine/Anomaly access via xlibs where wrappers exist.
- **Fail fast.** Guard clauses return result codes. No silent swallowing.

### Radiant evaluation

Radiant causes fire on A-Life state changes: smart transitions and player proximity. The callback's squad is both sensor and responder.

Two A-Life speeds. Off-map: round-robin across ~30 maps (going_speed), steady enter/leave_smart. On-map: frame-rate sim (current_level_going_speed), sparse transitions (few per minute). actor_on_smart adapter fills the on-map gap.

### Radiant vs reactive

| | Radiant | Reactive |
|-|---------|----------|
| Trigger | A-Life state change (smart transition, player proximity) | World event (death, medkit use, item pickup) |
| Callbacks | RADIANT_CALLBACKS (enter/leave/actor_on_smart) | Per-event (squad_on_npc_death, etc.) |
| Responder | Triggering squad IS the actor | Consequences find responders via find_squads_observed |
| Community filter | Cause's responsibility | Consequence's responsibility |

**Synthetic callbacks.** xevent hooks intercept game functions, emit callbacks with x_ prefix (xr_eat_medkit.consume_medkit -> x_npc_medkit_use). actor_on_hit_callback fires 50+/sec; item use ~0.6/min.

### Invariants

Not obvious from any single file. Violations cause subtle bugs.

1. **Serialization boundary.** m_data accepts only primitives and tables of primitives. Functions, userdata, metatables silently dropped on save. Arrival handler keys are strings in scripted_squads; handler functions live in _arrival_handlers and are re-created every load.

2. **Result code contract.** Every consequence returns { code = RESULT.X }. Consumer checks .code for chain propagation. Missing or malformed return = error.

3. **Scripted bypass.** scripted_target overrides SIMBOARD:get_squad_target(). Engine target_precondition (faction check, capacity check) is NOT evaluated for scripted squads. AP enforces faction safety in _build_filter and capacity safety in ap_utils.find_smart (unified capacity filter using real SIMBOARD population + in-transit count from _destination_counts).

4. **Mutation volatility.** Runtime smart terrain mutations (faction_controlled, respawn_params, faction) rebuilt from LTX on every load. Anything that mutates these must save/restore independently. Only conquered_smarts does this (two-phase).

5. **Unscriptable guard contract.** Named NPCs, traders, quest givers, task givers, and companions must never be scripted, moved, or modified by AP. Three layers enforce this:
   - **Cause level:** radiant causes (needs, stash, area) guard the triggering squad (it IS the responder). Reactive causes guard the entity AP would act on: elite guards the killer (gets buffs), elitekill guards the killer (gets bounty + hunters sent), wounded guards the patient (draws predators), harvest guards the taker (draws outlaws). Reactive causes where the trigger is a dead victim (massacre, squadkill, basekill) need no guard -- consequences find responders via find_squads.
   - **Consequence level:** all find_squads callers pass `exclude_unscriptable = true`. _find_hunters (elitekill_targeted) checks per hunter.
   - **Externally scripted level:** radiant causes (needs, stash, area) check is_externally_scripted(squad) before is_unscriptable_squad. find_squads callers pass exclude_externally_scripted = true. _find_hunters (elitekill_targeted) checks per hunter. Catches Guards Spawner, Warfare, LTX condlist, AP's own mid-mission squads.
   - **Tracker level:** script_squad and script_actor_target reject unscriptable + externally_scripted squads as defense-in-depth. Returns UNSCRIPTABLE_BYPASSED (WARN level) if reached -- indicates upstream filter gap.

---

## Pipeline Gates

Ordered gates in dispatch(). Gates 1-2 run before any predicate. Gates 3-4 are per-key.

| # | Name | Type | Scope | Setting |
|---|------|------|-------|---------|
| 1 | On/off-map ratio | Bresenham integer admission, level-ID comparison | shared | `alife_ratio` MCM (-10..+10, default 8) |
| 2a | Radiant pacer | Token bucket, single key "radiant" | global | `distributor_interval_sec` MCM (default 5) |
| 2b | Reactive pacer | Token bucket, per-callback-type key | per-type | `REACTIVE_EVENT_INTERVAL_SEC` (1s constant) |
| 3 | Cause budget | TTL counter, sliding window 60s | per-cause | `cause_max_events` MCM |
| 4 | Consequence budget | Token bucket, peek then acquire on success | per-consequence | `consequence_max_events` MCM, 60s window |

**Gate 1 (Bresenham).** Off-map outnumbers on-map ~50:1. Integer cross-multiplication for on:off ratio. At ratio=8: ~4:1 on:off. Formula: `throttled_count * |r| <= (10 - |r|) * favored_count`. Reset at 32768.

**Gate 2 (split).** Two xttltable.create_token_bucket instances, independent key spaces. Radiant and reactive never compete for tokens. Without the split, the adapter's ~6.6/sec radiant stream starves 100% of reactive events (measured: 356/356 blocked in 6-min playtest).

### Cooldowns and lifecycle TTLs

| Name | Type | Scope | Duration | Location |
|------|------|-------|----------|----------|
| Destination capacity | Reverse index (_destination_counts) | per-smart-id | Real-time: SIMBOARD.smarts[id].squads + in-transit count | ap_alife_tracker, ap_utils.find_smart |
| Hit modifier lock | Mutex (acquire_lock) | global | 1s | ap_object_mutator |
| Grenade restock cooldown | os.clock timestamp | per-NPC | MCM: elite_grenade_restock_cooldown_hours | ap_object_mutator |
| AP ammo restock cooldown | os.clock timestamp | per-NPC | MCM: elite_ap_ammo_cooldown_hours | ap_object_mutator |
| Scripted squad TTL | xtime.game_sec() timestamp | per-squad | 7200 game-seconds (SCRIPTED_SQUAD_TTL) | ap_alife_tracker |
| Elite dead grace | xttltable TTL table | per-entity | 3600s (DEAD_GRACE_SEC) | ap_alife_tracker |
| Elitekill victim cooldown | create_cooldown | per-victim | MCM: elitekill_cooldown_sec | ap_cause_elitekill |

---

## Result Codes

OUTCOME_PROPAGATION naming (see conventions.md). Name encodes outcome + chain behavior.

Stop codes: **OK_STOP** (success, exclusive), **ERROR_STOP** (failure, abort). Propagating codes (_NEXT suffix): **RULES_NEXT** (conditions unmet), **CHANCE_NEXT** (roll failed), **DISABLED_NEXT** (MCM off), **THROTTLED_NEXT** (rate limited).

**OK_NEXT** -non-exclusive success. Rare. Example: elitekill_bounty (p=0) pays the killer, then elitekill_targeted (p=10) sends hunters.

---

## Causes

| Cause | Type | Callback(s) | Conditions |
|-------|------|-------------|------------|
| MASSACRE | reactive | squad_on_npc_death | IsStalker(victim), at smart, not is_base, kill_count >= threshold |
| SQUADKILL | reactive | squad_on_npc_death | npc_count <= 1 (last member), not is_base |
| BASEKILL | reactive | squad_on_npc_death | IsStalker(victim), at is_base, kill_count >= threshold, smart has faction |
| ELITE | reactive | squad_on_npc_death | killer not unscriptable, killer_id exists, projected_kill_count crosses new elite level |
| ELITEKILL | reactive | squad_on_npc_death | is_elite(victim_id), killer not unscriptable, per-victim cooldown not active |
| WOUNDED | reactive | x_npc_medkit_use, actor_on_item_use | subject not unscriptable, subject exists, not is_base |
| HARVEST | reactive | actor_on_item_take, npc_on_item_take | IsArtefact(item), NPC taker not unscriptable |
| STASH | radiant | RADIANT_CALLBACKS | not unscriptable, stash within RADIUS_DISTANT_MAX |
| AREA | radiant | RADIANT_CALLBACKS | not unscriptable, community_stalker, empty smart, not is_base |
| NEEDS | radiant | RADIANT_CALLBACKS | not unscriptable, community_stalker, level.present(), Hull drive scoring (see Needs) |

### Predicate contract

Signature: `function(trace, ...callback_args) -> { cause = CAUSE.X, ...payload } | nil`

Producer wraps each call in observe(), attaches ._trace, publishes to xbus, increments cause counter. Predicates only evaluate and return. No observe(), no publish(), no trace creation, no counter manipulation -producer handles all of it.

---

## Consequence Patterns

Gate sequence per consequence: enabled -> chance -> rate limit -> logic -> PDA. Rush mode (max-speed movement) is per-consequence MCM toggle (`consequence_{name}_rush`, default true for tactical/outward, false for routine needs).

- **respond** -find nearby faction squads (50-500m), script toward event location. Used by: massacre_scavenge, massacre_investigate, basekill_support, wounded_hunt, wounded_help.
- **chase** -arrival-based pursuit loop. Each arrival re-evaluates: alive? same level? not co-located? under max_chases? Pass -> re-script to target's current position. Fail -> unscript. Player targets use script_actor_target (engine-native, no loop). Used by: squadkill_revenge, elitekill_targeted, harvest_hunt.
- **flee** -script to nearest friendly base, same level. Used by: squadkill_flee, basekill_flee.
- **state** -no movement. Update tracker, markers, rewards. Used by: elite_promote, elitekill_bounty.
- **stash** -script to stash's nearest smart. On-arrive: inventory ops via alife (loot/fill). Ambush is passive, no arrival handler. Base guard: squads at a base skip all stash consequences (won't leave base to chase stashes). Used by: stash_loot, stash_fill, stash_ambush.
- **conquest** -script to empty smart, conquer immediately on fire, not on arrival. Used by: area_conquer.
- **needs** -config-driven (15 rows in CONFIGS, _make_handler factory). Triggering squad IS the responder. Pipeline: need match -> enabled -> chance -> _build_filter -> find_smart(500m, capacity-filtered) -> script(rush per MCM) -> OK_STOP. On-arrive: consume/trade/passive. Gulag handles animations; AP never touches the job planner.

---

## Consequences

| Cause | Consequence | p | Pattern | Effect |
|-------|-------------|---|---------|--------|
| MASSACRE | massacre_scavenge | 10 | respond | find_squads(scavenger, 50-500m) -> script to massacre smart |
| MASSACRE | massacre_investigate | 20 | respond | find_squads(event faction, 50-500m) -> script to massacre smart |
| SQUADKILL | squadkill_revenge | 15 | chase | find_squads(victim faction, 50-500m) -> chase killer |
| SQUADKILL | squadkill_flee | 25 | flee | find_squads(victim faction, 50-500m) -> script to nearest base |
| BASEKILL | basekill_support | 15 | respond | find_squads(base factions, 50-500m) -> script to attacked base |
| BASEKILL | basekill_flee | 25 | flee | find_squads(base factions, 0-50m) -> script to distant base |
| ELITE | elite_promote | 15 | state | update_elite, map marker |
| ELITEKILL | elitekill_bounty | 0 | state | give_money (base * level +/-10%, capped) |
| ELITEKILL | elitekill_targeted | 10 | chase | get_elites_on_level -> chase killer with elite hunters |
| WOUNDED | wounded_hunt | 15 | respond | find_squads(predator, 50-500m) -> script to wounded's smart |
| WOUNDED | wounded_help | 25 | respond | find_squads(wounded faction, 50-500m) -> script to wounded's smart |
| HARVEST | harvest_hunt | 15 | chase | find_squads(harvest, 50-500m) -> chase artefact taker |
| STASH | stash_loot | 10 | stash | community_stalker -> script to stash, on_arrive: loot to NPC |
| STASH | stash_ambush | 20 | stash | community_ambusher -> script to stash smart (passive) |
| STASH | stash_fill | 30 | stash | community_stalker -> script to stash, on_arrive: fill stash |
| AREA | area_conquer | 20 | conquest | script to empty smart, conquer immediately |
| NEEDS | hunger_campfire | 10 | needs | has_campfire + exclude_enemy -> consume HUNGER |
| NEEDS | sleep_campfire | 10 | needs | has_campfire + exclude_enemy |
| NEEDS | rest_campfire | 10 | needs | has_campfire + exclude_enemy -> consume REST |
| NEEDS | heal_shelter | 10 | needs | has_surge_shelter + exclude_enemy -> consume HEAL |
| NEEDS | shelter_indoor | 10 | needs | has_surge_shelter + exclude_enemy |
| NEEDS | money_harvest | 10 | needs | has_anomaly |
| NEEDS | money_hunt | 20 | needs | is_lair |
| NEEDS | supply_trader | 10 | needs | has_trader_job -> trade exchange |
| NEEDS | job_outpost | 20 | needs | NOT is_base + NOT is_lair + exclude_enemy -> consume GUARD |
| NEEDS | job_explore | 30 | needs | unclaimed (fac="none") |
| NEEDS | job_research | 40 | needs | has_anomaly |
| NEEDS | job_worship | 15 | needs | NOT is_base + monolith/greh/zombied |
| NEEDS | job_exercise | 15 | needs | NOT is_base + army/dolg -> consume EXERCISE |
| NEEDS | social_campfire | 10 | needs | has_campfire + exclude_enemy -> consume SOCIAL |
| NEEDS | social_base | 20 | needs | is_base + exclude_enemy -> consume SOCIAL |

### _build_filter

One closure per handler call. 3 concerns evaluated per-smart in _build_filter (single pass, nearest match):

1. **Type filter** (entry.filter): has_campfire, has_surge_shelter, is_base, NOT is_base, NOT is_base + NOT is_lair, unclaimed (fac="none"), has_anomaly, is_lair, has_trader_job, nil
2. **Faction list** (entry.faction_list): xsmart.has_factions(smart, list) -worship, exercise
3. **Enemy exclusion** (entry.exclude_enemy): is_factions_enemies(community, smart_faction) -scripted_target bypasses engine target_precondition

Capacity filtering is handled by ap_utils.find_smart (unified, covers all consequence paths).

### Arrival dispatch

_on_arrive fires when squad reaches destination. Online/offline checked at arrival, not at trigger.

```
1. Always: reset DTO timestamp (args.dto_field)
2. Online check: db.storage[squad:commander_id()].object
   Online  -> _dispatch_arrive(go, args):
              args.section_key -> _consume(go, NEEDS_SECTIONS[key])
              args.arrive_fn == "trade" -> _arrive_trade(go)
   Offline -> no-op (no game_object)
```

DTO resets regardless -the squad traveled. Consume needs a game_object. Animations from vanilla gulag jobs (campfire_point, animpoint, walker).

- **consume KEY**: `_consume(go, NEEDS_SECTIONS[KEY])` -iterate sections, release first match via `alife_release_id`
- **trade exchange**: `_arrive_trade` -scan for `af_*` artefact, release it, create random supply item
- **--** (passive): gulag jobs handle behavior at destination
- Section lists: `ap_const.NEEDS_SECTIONS` (7 keys)

---

## Needs: Drive Scoring

Single predicate on RADIANT_CALLBACKS. Guards: community_stalker, level.present().

Nine needs, Maslow-weighted (SHELTER 5.0 down to SOCIAL 0.8). Hull drive: `drive = weight * (elapsed / threshold)^2`. Quadratic -2x threshold = 4x weight. Highest drive wins.

MCM per need: enabled gate (cause_{need}_enabled), threshold (HUNGER 8h, SLEEP 16h, REST 4h, SHELTER 24h, HEAL 12h, SUPPLY 48h, MONEY 24h, JOB 18h, SOCIAL 36h). SLEEP has night gate (game hours >= 20 or < 5).

Lazy init: first eval scatters 9 DTO timestamps across 0..2*threshold. Roughly half overdue on first check.

### DTO (stalker_needs in ap_alife_tracker)

- Per-squad game-second timestamps, 9 fields: `last_{need}_at`
- Time source: `game.get_game_time():diffSec(level.get_start_time())`
- API: get/init/reset via `ap_alife_tracker`
- Stale squads cleaned in `periodic_sync`

---

## Tracker (ap_alife_tracker)

### Data tables

| Table | Key | Content | Persistence |
|-------|-----|---------|-------------|
| killers | entity_id | kill count | save/load |
| elites | entity_id | { level, kills, level_id, name, is_stalker } | save/load |
| elite_dead | entity_id | same as elites (xttltable, 3600s grace after death) | transient |
| scripted_squads | squad_id | { scripted_target, target_smart_id, tracked_at, on_arrive, on_arrive_args } | save/load |
| conquered_smarts | smart_id | { faction, conquered_at } | save/load (two-phase, in ap_smart_mutator) |
| stalker_needs | squad_id | { last_hunger_at ... last_social_at } | save/load |

### Scripted squads

```
script_squad(squad, smart, opts) -> cleanup_stale -> validate -> release if re-script -> control_squad -> register
script_actor_target(squad)       -> register as "actor" -> target_actor (rush, no arrival detection)
unscript_squad(squad_or_id)      -> remove from table -> release_squad (clear scripted_target)
```

- `scripted_target` bypasses `SIMBOARD:get_squad_target()` -squad enters `specific_update` (direct A->B) instead of `generic_update` (simulation re-evaluation)
- `rush_to_target` sets run+danger behavior (via `xr_reach_task`)
- `scripted_target` persists across save/load natively (`sim_squad_scripted` STATE_Write/STATE_Read)
- TTL: 7200 game-seconds (`SCRIPTED_SQUAD_TTL`). Stale entries (entity gone or expired) cleaned before every write

### Arrival detection

_check_arrivals runs every 30s (CreateTimeEvent). For each scripted_squad with target_smart_id: xsmart.is_arrived(squad, smart). Arrived -> dispatch on_arrive handler, unscript. Actor targets have no arrival detection -engine handles pursuit.

### On-arrive handler registry

- `_arrival_handlers`: `{ [key] = function(squad, args) }`
- Registered via `register_arrival_handler(key, fn)` during `on_game_start`
- Keys/args persist (serializable). Functions are transient -re-registered every load

| Handler key | Registered by | On arrival |
|-------------|---------------|------------|
| stash_loot | stash_loot | loot stash to NPC inventory |
| stash_fill | stash_fill | fill stash with random items |
| revenge_chase | squadkill_revenge | re-script to killer's current smart |
| harvest_chase | harvest_hunt | re-script to taker's current smart |
| elitekill_chase | elitekill_targeted | re-script to killer's current smart |
| {15 need keys} | consequence_needs | reset DTO, online: _dispatch_arrive |

### Save/load

Tracker tables persisted to m_data.ap_alife_tracker (conquest separately in m_data.ap_smart_mutator). On load: tracked_at persists as game-time (survives save/load). Squads stay scripted. Handler functions re-registered on on_game_start. Conquered smarts use two-phase restore (see Runtime Smart Mutation).

---

## Chase API (ap_utils)

**chase_start(trace, chasers, target, pda_data, opts)** -target=nil -> script_actor_target (engine-native, no loop). target=squad -> script_squad with chase arrival handler.

**chase_register(opts)** -arrival handler. Each arrival: alive? same level? not co-located? under max_chases? Pass -> re-script. Exceed max -> unscript. Target gone -> PDA "lost".

---

## Runtime Smart Mutation

Engine reads smart config from LTX at load: faction_controlled, respawn_params, default_faction. AP mutates these at runtime for territory ownership. No disk writes.

### Flow

```
area_conquer consequence
  -> ap_alife_tracker.conquer_smart(smart_id, faction)
    -> xsmart.set_faction_controlled(smart, faction, spawn_num)
       smart.faction_controlled = entries for all 12 factions
       smart.respawn_params = fc_* entries (conqueror's squads)
       smart.faction = conqueror
```

- `set_faction_controlled` mirrors engine `read_params` (`smart_terrain.script`)
- Injects `faction_controlled` for all 12 factions, preserves `already_spawned` counts, sets `smart.faction`
- `try_respawn` then spawns conqueror's faction via standard logic
- `clear_faction_controlled` reverts all mutations, restores `default_faction`
- ZCP compatible -ZCP `try_respawn` uses the same `faction_controlled` check as vanilla
- Owned by `ap_smart_mutator.script` (extracted from ap_alife_tracker). Own save/load callbacks.

### FIFO eviction

- Max `area_conquest_max_smarts` (MCM, default 50)
- Full -> evict oldest `conquered_at`, clear mutations
- Monotonic sequence counter (`_conquest_seq`), not wall clock -survives save/load
- Same-faction re-conquest refreshes sequence (LRU)
- Different-faction overwrites without eviction

### Two-phase save/load

Mutations lost at STATE_Read (engine rebuilds from LTX). Two-phase restore:

1. **save_state** -conquered_smarts + _conquest_seq persisted to m_data
2. **load_state** -_conquered_pending = saved data, conquered_smarts = {} (cleared)
3. **STATE_Read** -engine rebuilds smart data from LTX (mutations gone)
4. **on_game_load** -re-apply set_faction_controlled from _conquered_pending (entities exist)
5. **periodic_sync** -remove stale smart IDs (entity destroyed)

---

## Runtime Combat Modifiers (ap_object_mutator)

npc_on_before_hit -> combat buffs from tracker kill data. Gated by acquire_lock (1s).

Level: `min(10, floor(kills / elite_kills_per_level))`

| Buff | Mechanism | Scaling | MCM |
|------|-----------|---------|-----|
| Grenades | create_item if none in inventory | count = level | elite_grenade_restock_enabled |
| AP ammo | Best k_ap ammo for equipped weapon | boxes = ceil(level/2) | elite_ap_ammo_enabled |
| Rank boost | set_character_rank(threshold) | threshold table | rank_boost_enabled |

Rank thresholds: L1-2=10000, L3-4=20000, L5-6=25000, L7-8=35000, L9-10=50000. One-way (never reduces). AP ammo: weapon-pack agnostic, cached per weapon section.

---

## Extension

**New cause:** add CAUSE.X to ap_const, create ap_cause_x.script with predicate, register via ap_producer_reactive.register(CAUSE.X, { callback = CALLBACK.Y }, fn). Radiant: register per RADIANT_CALLBACKS entry.

**New consequence:** add CONSEQUENCE.X to ap_const, create handler returning result codes, register via ap_consumer.register(CONSEQUENCE.X, { event = CAUSE.Y, priority = N }, fn). Gate order: enabled -> chance -> logic. OK_STOP = exclusive, _NEXT = propagate. Arrival behavior: register_arrival_handler(key, fn) in on_game_start.

**New radiant callback:** add to ap_const.RADIANT_CALLBACKS (auto-registered by all radiant causes). Token bucket capacity=1 -anything over ~15/sec is wasted. Current three: enter/leave_smart (steady off-map, sparse on-map), actor_on_smart (on-map density).
