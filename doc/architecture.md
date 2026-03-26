# AlifePlus Architecture

Behavior mod for STALKER Anomaly. Observes A-Life events, detects patterns (causes), triggers squad behaviors (consequences). Built on xlibs.

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

AP runs inside the X-Ray Monolith engine, hosted by the Anomaly mod framework. It does not patch engine code or modify files on disk. All integration is through runtime callbacks, Lua globals, and xlibs wrappers.

| System | What AP uses | Interface |
|--------|-------------|-----------|
| X-Ray callbacks | squad_on_npc_death, enter/leave_smart, actor_on_interaction, npc_on_before_hit, actor_on_item_use/take, npc_on_item_take, save/load_state, on_game_load, actor_on_first_update | RegisterScriptCallback |
| X-Ray A-Life | alife(), alife_object, alife_release_id, alife_create_item | Engine globals |
| X-Ray game state | db.storage, db.actor, level.present(), level.get_start_time(), level.get_time_hours(), game.get_game_time() | Engine globals |
| Anomaly simulation | Squad targeting via scripted_target, rush_to_target | sim_squad_scripted (via xsquad) |
| Anomaly smart_terrain | faction_controlled, respawn_params, default_faction, try_respawn | Runtime table mutation (via xsmart) |
| Anomaly scripts | xr_reach_task (run+danger behavior), xr_eat_medkit (NPC medkit hook) | xevent.hook, direct reference |
| Anomaly game_relations | is_factions_enemies | Direct call |
| MCM (ui_mcm) | Settings UI, user configuration | xmcm.create_getter |
| Save system | m_data table in save_state/load_state callbacks | Anomaly serialization |
| xlibs | xbus, xsmart, xsquad, xcreature, xlevel, xlog, xpda, xmath, xtable, xttltable, xevent, xmcm, xinspect | Lua require (version-gated by _ap_deps) |

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
| ap_alife_tracker | State manager: killers, elites, scripted_squads, conquered_smarts, stalker_needs |
| ap_alife_behavior | Elite combat buffs: grenades, AP ammo, rank boost (npc_on_before_hit) |
| ap_utils | Event wrappers, PDA, destination limiter, chase API, find helpers |
| ap_cause_* | Cause predicates (one file per cause group) |
| ap_consequence_* | Consequence handlers (one file per cause; needs = config table) |
| ap_messages | PDA message templates |
| ap_test | In-game debug commands: force needs, spawn squads, trigger causes |

---

## Lifecycle

AP initializes in four phases. Each phase has a strict precondition -- the engine guarantees the previous phase completed before the next begins.

### Phase 0 -- Module load

Lua require resolves all script files. Module-level code runs: engine globals cached to locals, constant tables built, xlibs API references captured. No game state exists yet.

### Phase 1 -- on_game_start

The engine calls on_game_start() on every script file. Execution order across files is undefined, but all operations within this phase are independent of each other:

- **_ap_deps** asserts xlibs installed and version == 1.2.0. Hard crash on mismatch.
- **ap_mcm** loads config from MCM defaults, registers on_option_change.
- **ap_debug** sets log level from MCM, registers on_option_change.
- **ap_alife_tracker** creates elite_dead TTL table, registers save_state/load_state/on_game_load/squad_on_npc_death, creates arrival check timer (30s) and optional periodic_sync timer.
- **ap_alife_behavior** registers npc_on_before_hit.
- **ap_utils** registers actor_on_first_update for marker cleanup.
- **ap_producer_reactive** resets dispatch state. Registers actor_on_first_update -> subscribe_to_callbacks. Does NOT subscribe to engine callbacks yet.
- **ap_consumer** registers actor_on_first_update -> _subscribe. Does NOT subscribe to xbus yet.
- **All ap_cause_*** register predicates with the producer via register().
- **All ap_consequence_*** register handlers with the consumer via register(). Register arrival handlers with tracker where needed.

After Phase 1: all predicates and handlers are registered, but no engine callbacks or xbus subscriptions are active.

### Phase 2 -- actor_on_first_update

Fires once the game world exists and the actor is loaded.

- **Producer** subscribes to RegisterScriptCallback for each unique callback type, plus actor_on_interaction for the radiant adapter.
- **Consumer** calls xbus.subscribe for each cause event type.
- **ap_debug** dumps current MCM settings to log.
- **ap_utils** clears ephemeral squad markers from previous session.

### Phase 3 -- on_game_load

Fires after the engine rebuilds all game objects from the save file (after STATE_Read). Runtime smart terrain mutations were lost during STATE_Read. The tracker re-applies conquered smart mutations from _conquered_pending (two-phase restore).

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

OTel-inspired hierarchical tracing via observe() (ap_debug). Each trace carries:

- **tid** -- Monotonic trace ID. Grep `tid=N` in alifeplus.log to follow a full cause->consequence chain across all log lines.
- **path** -- Slash-separated span hierarchy (e.g. `CAUSE.NEEDS/ACTION.FIND_DESTINATION/ACTION.MOVE_SQUAD`). Each observe(trace, op_name, fn) pushes a segment, times execution, logs result with path + timing, then pops.

Log format: `[OP_LABEL] [tid=N path=A/B] code:reason key=val [Xms]`

Producer creates the root trace for cause predicates. Consumer passes it through to consequences via event_data._trace. Predicates and handlers extend the hierarchy by calling observe(trace, ACTION.X, fn).

When log_level < DEBUG: observe() is a straight pass-through -- fn(trace, ...) called directly, no logging, no timing. _null_trace and _null_timer are null-object singletons with no-op methods. Overhead: one table lookup + one no-op call per observe() invocation.

---

## Design Principles

- **YANS (You Are Not Special).** Same rules for player and NPCs.
- **Low idle cost.** Event-driven pipeline. Two timers: arrival check (30s) and periodic cleanup.
- **Decoupled.** Causes publish to xbus. Consequences subscribe. Neither references the other. All engine/Anomaly access through xlibs wrappers where one exists.
- **Fail fast.** Guard clauses return result codes. No silent swallowing.

### Radiant evaluation

Radiant causes evaluate on A-Life state changes: squad arrivals and departures at smart terrains, and player proximity to occupied smarts. The evaluation trigger is decoupled from what is evaluated -- the callbacks carry a squad reference that becomes both the sensor (what observes the world) and the actor (what responds). One evaluation, one actor, from one independent event.

The engine processes A-Life at two speeds. Off-map: distant levels via round-robin (going_speed). enter/leave_smart fire at a steady, moderate rate across ~30 maps -- a monotone stream, spread thin. On-map: the player's level at frame rate (current_level_going_speed). enter/leave_smart fire only when squads physically transition between smarts -- bursty but infrequent (a few per minute on an active level). actor_on_smart (adapter from actor_on_interaction) provides evaluation density where the player is, compensating for the sparse on-map transition rate.

This replaces the god-loop pattern (centralized system scanning all smarts every N seconds). O(1) per event, frequency tracks natural A-Life activity.

### Radiant vs reactive

| | Radiant | Reactive |
|-|---------|----------|
| Trigger | A-Life state change (smart transition, player proximity) | World event (death, medkit use, item pickup) |
| Callbacks | RADIANT_CALLBACKS (enter/leave/actor_on_smart) | Per-event (squad_on_npc_death, etc.) |
| Responder | Triggering squad IS the actor | Consequences find responders via find_squads_observed |
| Community filter | Cause's responsibility | Consequence's responsibility |

**Synthetic callbacks.** Engine lacks some needed events. xevent hooks intercept game functions and emit synthetic callbacks with x_ prefix (e.g. xr_eat_medkit.consume_medkit -> x_npc_medkit_use). actor_on_hit_callback fires 50+/sec during damage; item use fires ~0.6/min.

### Invariants

These rules are not obvious from reading any single file. Violating them causes subtle bugs.

1. **Serialization boundary.** m_data accepts only primitives and tables of primitives. Functions, userdata, and metatables are silently dropped on save. Arrival handler keys are strings stored in scripted_squads; handler functions are registered separately in _arrival_handlers and re-created every load.

2. **Result code contract.** Every consequence handler returns a table with a .code field from RESULT. The consumer checks .code to decide chain propagation. A missing or malformed return is an error.

3. **Scripted bypass.** scripted_target overrides SIMBOARD:get_squad_target(). The engine's target_precondition (faction compatibility check) is NOT evaluated for scripted squads. AP must enforce faction safety itself -- this is why _build_filter has the faction_enemy concern.

4. **Mutation volatility.** Runtime smart terrain mutations (faction_controlled, respawn_params, faction) are rebuilt from LTX on every load. Any system that mutates these at runtime must implement its own save/restore. Currently only conquered_smarts does this (two-phase).

---

## Pipeline Gates

Ordered gates in dispatch(). Every event passes through gates 1-3 before any predicate runs. Gates 4-5 are per-key.

| # | Name | Type | Scope | Setting |
|---|------|------|-------|---------|
| 1 | Level classification | Level-ID comparison | per-event | -- (classifies on/off-map) |
| 2 | On/off-map ratio | Bresenham integer admission | shared | `alife_ratio` MCM (-10..+10, default 8) |
| 3a | Radiant pacer | Token bucket, single key "radiant" | global | `distributor_interval_sec` MCM (default 3) |
| 3b | Reactive pacer | Token bucket, per-callback-type key | per-type | `REACTIVE_EVENT_INTERVAL_SEC` (1s constant) |
| 4 | Cause budget | TTL counter, sliding window 60s | per-cause | `cause_max_events` MCM |
| 5 | Consequence budget | Token bucket, peek then acquire on success | per-consequence | `consequence_max_events` MCM, 60s window |

**Gate 2 (Bresenham).** Off-map events outnumber on-map ~50:1 across 30 maps. The gate uses Bresenham-derived integer cross-multiplication to enforce a configurable on:off ratio without floating-point division. At ratio=8: ~4:1 on:off. Formula: `throttled_count * |r| <= (10 - |r|) * favored_count`. Counters reset at 32768.

**Gate 3 (split).** Two separate xttltable.create_token_bucket instances with independent key spaces. Radiant and reactive events never compete for tokens. Without this split, the adapter's ~6.6/sec radiant stream starves 100% of reactive events (measured: 356/356 blocked in 6-min playtest).

### Cooldowns and Lifecycle TTLs

| Name | Type | Scope | Duration | Location |
|------|------|-------|----------|----------|
| Destination limiter | TTL counter | per-smart-id | ap_const: DESTINATION_LOCK_MAX, DESTINATION_LOCK_TTL_SEC | ap_utils |
| Hit modifier lock | Mutex (acquire_lock) | global | 1s | ap_alife_behavior |
| Grenade restock cooldown | os.clock timestamp | per-NPC | MCM: elite_grenade_restock_cooldown_hours | ap_alife_behavior |
| AP ammo restock cooldown | os.clock timestamp | per-NPC | MCM: elite_ap_ammo_cooldown_hours | ap_alife_behavior |
| Scripted squad TTL | os.clock timestamp | per-squad | 7200s (SCRIPTED_SQUAD_TTL) | ap_alife_tracker |
| Elite dead grace | xttltable TTL table | per-entity | 3600s (DEAD_GRACE_SEC) | ap_alife_tracker |
| Elitekill victim cooldown | create_cooldown | per-victim | MCM: elitekill_cooldown_sec | ap_cause_elitekill |

---

## Result Codes

Result codes follow the OUTCOME_PROPAGATION naming pattern (see conventions.md). The name encodes both outcome and chain behavior.

Two codes halt the consequence chain: **OK_STOP** (success, exclusive -- one consequence claimed the event) and **ERROR_STOP** (failure, abort). Everything else carries a _NEXT suffix and propagates, letting lower-priority consequences try: **RULES_NEXT** (conditions unmet), **CHANCE_NEXT** (roll failed), **DISABLED_NEXT** (MCM gate off), **THROTTLED_NEXT** (rate limited).

**OK_NEXT** is non-exclusive success -- rare, used for side effects that don't preclude other handlers (e.g. elitekill_bounty at p=0 pays the killer, then elitekill_targeted at p=10 sends hunters).

The consumer dispatches consequences by priority (lower = first) and breaks on any _STOP code.

---

## Causes

| Cause | Type | Callback(s) | Conditions |
|-------|------|-------------|------------|
| MASSACRE | reactive | squad_on_npc_death | IsStalker(victim), at smart, not is_base, kill_count >= threshold |
| SQUADKILL | reactive | squad_on_npc_death | npc_count <= 1 (last member), not is_base |
| BASEKILL | reactive | squad_on_npc_death | IsStalker(victim), at is_base, kill_count >= threshold, smart has faction |
| ELITE | reactive | squad_on_npc_death | killer_id exists, projected_kill_count crosses new elite level |
| ELITEKILL | reactive | squad_on_npc_death | is_elite(victim_id), per-victim cooldown not active |
| WOUNDED | reactive | x_npc_medkit_use, actor_on_item_use | subject exists, not is_base |
| HARVEST | reactive | actor_on_item_take, npc_on_item_take | IsArtefact(item) |
| STASH | radiant | RADIANT_CALLBACKS | stash within RADIUS_DISTANT_MAX |
| AREA | radiant | RADIANT_CALLBACKS | community_stalker, empty smart, not is_base |
| NEEDS | radiant | RADIANT_CALLBACKS | community_stalker, level.present(), Hull drive scoring (see Needs) |

### Predicate contract

Signature: `function(trace, ...callback_args) -> { cause = CAUSE.X, ...payload } | nil`

**Producer responsibilities** (ap_producer_reactive):
- Subscribe to X-Ray callbacks via RegisterScriptCallback
- Create root trace, wrap each predicate call in observe()
- If predicate returns a table with .cause, attach trace as ._trace and publish to xbus
- Increment cause rate counter after publish

**Predicate responsibilities** (ap_cause_*.script):
- Return { cause = CAUSE.X, ...payload } on match, or nil on no-match
- No inner observe() -- producer already wraps the call
- No event_publish() -- producer handles publishing
- No trace creation -- producer provides the trace as first argument
- No rate counter manipulation -- producer increments after publish

---

## Consequence Patterns

Consequences share a common gate sequence (enabled MCM -> chance MCM -> per-consequence rate limit -> business logic -> PDA) but diverge into distinct architectural patterns:

**Respond-to-event.** Reactive consequences find nearby squads matching faction criteria and script them toward the event location. The event provides "where" and "what happened"; the consequence finds "who" and decides "how." Used by: massacre (scavenge, investigate), basekill (support), wounded (hunt, help).

**Chase.** A pursuit loop built on the arrival handler system. chase_start scripts chasers toward the target's current position. On arrival, the chase handler re-evaluates: target alive? Same level? Not co-located? Under max_chases? If all pass, re-script to target's new position. If any fail, unscript. Player targets use engine-native pursuit (script_actor_target) with no chase loop -- the engine handles following. Used by: squadkill_revenge, elitekill_targeted, harvest_hunt.

**Flee-to-safety.** Find nearest friendly base on the same level, script squad there. Inverse of respond-to-event -- move away from danger, not toward it. Used by: squadkill_flee, basekill_flee.

**State mutation.** No squad movement. Update tracker state, create markers, give rewards. Used by: elite_promote (update_elite + marker), elitekill_bounty (give_money).

**Stash interaction.** Script to stash's nearest smart. On-arrive handler manipulates stash inventory via alife operations (loot or fill). stash_ambush is passive (no arrival handler -- squad just occupies the smart). Used by: stash_loot, stash_fill, stash_ambush.

**Area conquest.** Script to empty claimable smart. Mutates respawn parameters immediately on consequence fire, not on arrival (see Runtime Smart Mutation). Used by: area_conquer.

**Needs template.** Config-table-driven. 15 rows in CONFIGS, same _make_handler(entry) factory. The triggering squad IS the responder (radiant pattern). Find destination smart by composed filter, script the squad, on-arrive consume items or trade (online only) or do nothing (gulag jobs handle behavior). All 15 follow identical code path; variation is pure data in config rows (filter, faction, section_key). The engine's gulag system assigns animations by smart type -- AP never touches the job planner.

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
| NEEDS | hunger_campfire | 10 | needs | has_campfire -> consume HUNGER |
| NEEDS | sleep_campfire | 10 | needs | has_campfire |
| NEEDS | rest_campfire | 10 | needs | has_campfire -> consume REST |
| NEEDS | heal_base | 10 | needs | is_base + faction_filter -> consume HEAL |
| NEEDS | shelter_indoor | 10 | needs | is_base + faction_filter |
| NEEDS | money_search | 10 | needs | has_anomaly |
| NEEDS | money_hunt | 20 | needs | is_lair |
| NEEDS | supply_trader | 10 | needs | has_trader_job -> trade exchange |
| NEEDS | job_guard | 20 | needs | is_base + faction_filter -> consume GUARD |
| NEEDS | job_explore | 30 | needs | -- |
| NEEDS | job_research | 40 | needs | has_anomaly |
| NEEDS | job_worship | 15 | needs | is_base + monolith/greh/zombied |
| NEEDS | job_exercise | 15 | needs | is_base + army/dolg -> consume EXERCISE |
| NEEDS | social_campfire | 10 | needs | has_campfire -> consume SOCIAL |
| NEEDS | social_base | 20 | needs | is_base + faction_filter -> consume SOCIAL |

### Needs implementation

All 15 NEEDS rows live in ap_consequence_needs.script. Config table CONFIGS, _make_handler(entry) builds the closure. Template: need match -> enabled -> chance -> resolve squad -> _build_filter -> find_smart(max 500m) -> script_squad(rush) -> mark_destination -> OK_STOP.

**faction_filter** = is_factions_enemies(community, smart_faction) -- prevents scripting to hostile bases. Named factions (monolith/greh/zombied, army/dolg) use has_factions(smart, faction_list).

**consume KEY** = _consume(go, NEEDS_SECTIONS[KEY]) via npc:object per section, release first match via alife_release_id. **trade exchange** = _arrive_trade(go) -- scan inventory for af_* artefact, release it, create random supply item. **--** = passive (gulag jobs handle behavior at destination). Section lists in ap_const.NEEDS_SECTIONS (7 keys).

### _build_filter

Composes one closure per handler invocation. 4 concerns, all evaluated per-smart inside find_smart (one filter, one pass, nearest match returned):

1. **Type filter** (entry.filter): has_campfire, is_base, has_anomaly, is_lair, has_trader_job, or nil
2. **Faction list** (entry.faction_list): xsmart.has_factions(smart, list) -- worship, exercise
3. **Faction enemy** (entry.faction_filter): is_factions_enemies(community, smart_faction) -- prevents AP from scripting a squad to a hostile base (scripted_target bypasses the engine's target_precondition check)
4. **Destination capacity** (check_destination): TTL counter, max N squads per smart within TTL window (ap_const: DESTINATION_LOCK_MAX, DESTINATION_LOCK_TTL_SEC)

### Arrival dispatch

_on_arrive(squad, args) fires when the squad reaches its destination smart. Online/offline checked at arrival, not at trigger -- a squad scripted while online may arrive offline or vice versa.

```
1. Always: reset DTO timestamp (args.dto_field)
2. Online check: db.storage[squad:commander_id()].object
   Online  -> _dispatch_arrive(go, args):
              args.section_key -> _consume(go, NEEDS_SECTIONS[key])
              args.arrive_fn == "trade" -> _arrive_trade(go)
   Offline -> no-op (no game_object, no inventory)
```

DTO always resets (the need was addressed -- the squad traveled). Consume only fires online (inventory operations require a game_object). Animations at destination are handled by vanilla gulag jobs (campfire_point, animpoint, walker). AP does NOT touch the job planner.

---

## Needs: Drive Scoring

Single predicate on RADIANT_CALLBACKS. Guards: community_stalker, level.present().

Nine needs, Maslow-weighted from SHELTER (5.0) down to SOCIAL (0.8). Hull drive formula: `drive = weight * (elapsed / threshold)^2`. Quadratic: a need at 2x its threshold scores 4x its base weight. The highest drive wins.

Each need has an MCM-configurable enabled gate (cause_{need}_enabled) and threshold (HUNGER 8h, SLEEP 16h, REST 4h, SHELTER 24h, HEAL 12h, SUPPLY 48h, MONEY 24h, JOB 18h, SOCIAL 36h). SLEEP has an additional night gate (game hours >= 20 or < 5). Disabled needs are skipped before scoring.

Lazy init: first evaluation for a squad scatters all 9 DTO timestamps randomly across 0..2*threshold. Roughly half will already be overdue, producing a drive on the first check. Subsequent evaluations use Hull scoring normally.

### DTO (stalker_needs in ap_alife_tracker)

Per-squad game-second timestamps. 9 fields: last_{need}_at. Time source: game.get_game_time():diffSec(level.get_start_time()). API: get/init/reset via ap_alife_tracker. Stale squads cleaned in periodic_sync.

---

## Tracker (ap_alife_tracker)

Central state manager for all persistent AP data.

### Data tables

| Table | Key | Content | Persistence |
|-------|-----|---------|-------------|
| killers | entity_id | kill count | save/load |
| elites | entity_id | { level, kills, level_id, name, is_stalker } | save/load |
| elite_dead | entity_id | same as elites (xttltable, 3600s grace after death) | transient |
| scripted_squads | squad_id | { scripted_target, target_smart_id, tracked_at, on_arrive, on_arrive_args } | save/load |
| conquered_smarts | smart_id | { faction, conquered_at } | save/load (two-phase) |
| stalker_needs | squad_id | { last_hunger_at ... last_social_at } | save/load |

### Scripted squads

```
script_squad(squad, smart, opts) -> cleanup_stale -> validate -> release if re-script -> control_squad -> register
script_actor_target(squad)       -> register as "actor" -> target_actor (rush, no arrival detection)
unscript_squad(squad_or_id)      -> remove from table -> release_squad (clear scripted_target)
```

scripted_target bypasses SIMBOARD:get_squad_target() -- the squad enters specific_update (direct A->B) instead of generic_update (simulation re-evaluation with gulag reassignment). rush_to_target sets run+danger via xr_reach_task (src: xr_reach_task.script:300-313). scripted_target persists across save/load natively (src: sim_squad_scripted.script STATE_Write:940 / STATE_Read:984).

TTL: 7200s (SCRIPTED_SQUAD_TTL). Stale entries (entity gone or TTL expired) cleaned before every write.

### Arrival detection

_check_arrivals runs every 30s (CreateTimeEvent). For each scripted_squad with target_smart_id: xsmart.is_arrived(squad, smart). If arrived: dispatch on_arrive handler by key, then unscript. Actor targets have no arrival detection -- engine handles pursuit natively.

### On-arrive handler registry

_arrival_handlers: { [key] = function(squad, args) }. Consequences call register_arrival_handler(key, fn) during on_game_start. Keys and args in scripted_squads persist across save/load (serializable strings/tables). Functions are transient -- re-registered every load.

| Handler key | Registered by | On arrival |
|-------------|---------------|------------|
| stash_loot | stash_loot | loot stash to NPC inventory |
| stash_fill | stash_fill | fill stash with random items |
| revenge_chase | squadkill_revenge | re-script to killer's current smart |
| harvest_chase | harvest_hunt | re-script to taker's current smart |
| elitekill_chase | elitekill_targeted | re-script to killer's current smart |
| {15 need keys} | consequence_needs | reset DTO, online: _dispatch_arrive |

### Save/load

All tables persisted to m_data.ap_alife_tracker. On load: tracked_at reset to os.clock(). Squads stay scripted. Handler functions re-registered on on_game_start. Conquered smarts use two-phase restore (see Runtime Smart Mutation).

---

## Chase API (ap_utils)

Standardized pursuit for NPC and player targets.

**chase_register(opts).** Arrival handler. On each arrival: target alive? Same level? Not co-located? Under max_chases? If all pass, re-script to target's current smart. Exceeds max -> unscript. Target gone -> PDA "lost".

**chase_start(trace, chasers, target, pda_data, opts).** Initial script. target=nil -> script_actor_target (player, engine-native pursuit, no chase loop). target=squad -> script_squad with on_arrive chase handler (re-script loop).

Used by: squadkill_revenge, elitekill_targeted, harvest_hunt.

---

## Runtime Smart Mutation

The engine reads smart terrain configuration from LTX at load time: faction_controlled, respawn_params, default_faction. These tables determine which factions spawn at which smarts. AP mutates these runtime tables to change territory ownership dynamically, without modifying any files on disk.

### Flow

```
area_conquer consequence
  -> ap_alife_tracker.conquer_smart(smart_id, faction)
    -> xsmart.set_faction_controlled(smart, faction, spawn_num)
       smart.faction_controlled = entries for all 12 factions
       smart.respawn_params = fc_* entries (conqueror's squads)
       smart.faction = conqueror
```

set_faction_controlled mirrors the engine's own read_params (src: smart_terrain.script:246-267). It injects faction_controlled entries for all 12 factions, preserves the engine's already_spawned counts, and sets smart.faction. The engine's try_respawn then spawns the conqueror's faction using standard logic. clear_faction_controlled reverts all mutations and restores smart.faction to smart.default_faction.

ZCP compatible -- ZCP's try_respawn (src: line 1672) uses the identical faction_controlled check as vanilla.

### FIFO eviction

Max area_conquest_max_smarts (MCM, default 50). When full, the oldest conquered_at entry is evicted and its mutations cleared. Uses a monotonic sequence counter (_conquest_seq), not wall clock -- survives save/load without timestamp normalization. Same-faction re-conquest refreshes the sequence (LRU behavior). Different-faction overwrites the entry without eviction.

### Two-phase save/load

Smart terrain runtime tables are rebuilt from LTX on every load -- mutations are lost at STATE_Read. AP uses a two-phase restore:

1. **save_state** -- conquered_smarts + _conquest_seq persisted to m_data
2. **load_state** -- _conquered_pending = saved data, conquered_smarts = {} (cleared)
3. **STATE_Read** -- engine rebuilds smart data from LTX (mutations gone)
4. **on_game_load** -- re-apply set_faction_controlled from _conquered_pending (entities exist now)
5. **periodic_sync** -- remove stale smart IDs (entity destroyed)

---

## Runtime Combat Modifiers (ap_alife_behavior)

Intercepts npc_on_before_hit to apply combat buffs based on tracker kill data. Gated by acquire_lock (1s throttle per call).

Level formula: `min(10, floor(kills / elite_kills_per_level))`

| Buff | Mechanism | Scaling | MCM |
|------|-----------|---------|-----|
| Grenades | create_item if none in inventory | count = level | elite_grenade_restock_enabled |
| AP ammo | Best k_ap ammo for equipped weapon | boxes = ceil(level/2) | elite_ap_ammo_enabled |
| Rank boost | set_character_rank(threshold) | threshold table | rank_boost_enabled |

Rank thresholds: L1-2=10000, L3-4=20000, L5-6=25000, L7-8=35000, L9-10=50000. One-way (never reduces). AP ammo: weapon-pack agnostic, cached per weapon section.

---

## Extension Points

### Adding a cause

1. Add CAUSE.X to ap_const.CAUSE
2. Create ap_cause_x.script with a predicate following the predicate contract
3. In on_game_start, register: ap_producer_reactive.register(CAUSE.X, { callback = CALLBACK.Y }, predicate_fn)
4. For radiant causes: register once per entry in ap_const.RADIANT_CALLBACKS

### Adding a consequence

1. Add CONSEQUENCE.X to ap_const.CONSEQUENCE
2. Create ap_consequence_x.script with a handler returning result codes
3. In on_game_start, register: ap_consumer.register(CONSEQUENCE.X, { event = CAUSE.Y, priority = N }, handler_fn)
4. Standard gate order in handler: enabled MCM -> chance MCM -> business logic. Return OK_STOP to claim exclusively, _NEXT to let others try.
5. If arrival behavior needed: ap_alife_tracker.register_arrival_handler(key, fn) in on_game_start

### Adding a radiant callback

Add the callback string to ap_const.RADIANT_CALLBACKS. All radiant causes automatically register on it via their on_game_start loop. Zero cause file edits.

Choose callbacks with measured sustained frequency under 15 events/second (ideally ~10/minute). The radiant token bucket has capacity=1 -- high-frequency callbacks (actor_on_hit at 50+/sec, actor_on_update at 60/sec) will be dropped immediately with no benefit. The three current callbacks have complementary profiles: enter/leave_smart provide steady off-map coverage and sparse on-map transitions; actor_on_smart (adapter) fills the on-map gap with proximity-based evaluation density. A good radiant callback represents a meaningful A-Life state change, not a per-frame update.
