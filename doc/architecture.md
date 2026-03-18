# AlifePlus Architecture

Event-driven emergent gameplay system for STALKER Anomaly.

See `conventions.md` for naming rules, standard patterns, MCM settings, and logging format.

---

## Overview

AlifePlus observes A-Life simulation events, detects meaningful patterns (causes), and triggers reactive behaviors (consequences). Built entirely on xlibs abstractions.

```
  ENGINE CALLBACKS                 xevent HOOKS
  +-----------------------+        +------------------------+
  | squad_on_npc_death    |        | xr_eat_medkit hook     |
  | squad_on_enter_smart  |        |   -> x_npc_medkit_use  |
  | squad_on_leave_smart  |        +------------+-----------+
  +-----------+-----------+                     |
              |                                 |
              +----------------+----------------+
                               |
                               v
               +-------------------------------+
               |           PRODUCER            |
               | 1. distributor (budget/type)  |
               | 2. resolve_on_map (level cmp) |
               | 3. ratio_gate (Bresenham)     |
               | 4. class lock (on/off-map)    |
               | 5. per-cause rate limit       |
               | 6. predicate evaluation       |
               +---------------+---------------+
                               |
                               v
               +-----------------------------+
               |            xbus             |
               | cause:massacre, cause:stash |
               | cause:wounded, ...          |
               +--------------+--------------+
                              |
                              v
               +-----------------------------+
               |          CONSUMER           |
               | - priority dispatch         |
               | - OK_STOP/ERROR_STOP stops  |
               +--------------+--------------+
                              |
              +---------------+---------------+
              v               v               v
        [scavenge]    [investigate]     [revenge]
            p=10           p=20            p=15
```

---

## Core Concepts

| Term | Definition |
|------|------------|
| Cause | Engine callback + world-state predicate -> meaningful situation or nil |
| Consequence | Handler that subscribes to a cause event, executes business logic + side effects |
| Predicate | Pure function: `(trace, ...args) -> { cause, ...payload }` or `nil` |
| Producer | Dispatches X-Ray callbacks to registered predicates, publishes results to xbus |
| Distributor | Per-callback-type rate limiter inside producer (radiant gets 2x budget) |
| Consumer | Receives cause events from xbus, dispatches to consequences by priority |
| xbus | Pub/sub event bus. Causes publish, consequences subscribe. Zero coupling. |

---

## Design Principles

### 1. YANS (you are not special)

Nothing is player-centric. Same rules apply to player and NPCs. What happens to player happens to NPCs.

### 2. Zero Performance Impact

Even 1 frame dropped out of 60 is unacceptable. No god loops, no heavy iterations.

### 3. Event-Driven

Zero cost when idle. Callbacks trigger work only when events occur.

### 4. Radiant Evaluation, Not God Loop

Radiant causes evaluate on smart terrain transitions -- the natural flow of A-Life. The simulation itself determines when and how often evaluation happens.

| God Loop       | Radiant Evaluation        |
|----------------|---------------------------|
| System checks all smarts every 300s | Squad evaluates on smart terrain transition |
| System searches for squads to respond | The evaluating squad acts directly |
| Centralized, omniscient, gamey | Distributed, local, organic |
| O(n) cost per interval | O(1) cost per event |
| Fixed frequency regardless of activity | Self-regulating: follows natural A-Life flow |

#### Radiant cause flow

Each radiant cause registers for both `squad_on_enter_smart` and `squad_on_leave_smart`. Two evaluation moments per smart terrain transition.

```
Squad S transitions at smart
  -> Cause predicate: evaluate surroundings
  -> Publishes: { cause, squad_id = S, target = T, position, level_id }
  -> Consequences compete (priority + OK_STOP)
  -> Winner directs S to T
```

The cause does not know which callback fired. `smart` is the evaluation point. The triggering squad is the sensor AND the actor.

One evaluation -> one actor. Multiple actors from multiple independent transitions.

#### Reactive vs radiant

Both are event-driven -- no timers. The difference is what they react to.

**Reactive causes** fire on specific world events (death, medkit use, item pickup). Consequences use find_squads_observed to locate responders nearby.

**Radiant causes** fire on smart terrain transitions. The evaluating squad sees its surroundings and acts directly. The evaluating squad IS the responder -- no find_squads_observed.

### 5. Decoupled

Causes publish to xbus. Consequences subscribe to xbus. Neither references the other.

### 6. xlibs First

All side effects through xlibs wrappers, never raw X-Ray APIs.

### 7. Fail Fast

Guard clauses at the top of every predicate and handler. Missing data returns `{ code = RESULT.RULES_NEXT, reason = REASON.X }` immediately. Result codes propagate to consumer and get logged. No silent swallowing, no fallback state.

---

## Event Flow

1. X-Ray callback fires
2. Producer: distributor check (per-callback budget, blocked -> skip all, log)
3. Producer: resolve on-map (compare event level to actor level)
4. Producer: ratio gate (Bresenham admission control, blocked -> skip all, log)
5. Producer: class lock (on-map or off-map cooldown, blocked -> skip all, log)
6. Producer: for each registered predicate:
   a. Per-cause rate limit check (blocked -> skip, log THROTTLED_NEXT)
   b. Wrap predicate call in `observe()`, pass trace
   c. Predicate does filtering, returns `{ cause, ...data }` or nil
   d. If nil -> skip, log RULES_NEXT
   e. Increment rate limit counter
   f. Publish to xbus with trace attached
7. Consumer receives event from xbus
8. Consumer: for each consequence (by priority):
   a. Per-consequence rate limit check (blocked -> skip, log THROTTLED_NEXT)
   b. Call handler with event_data (trace available via event_data._trace)
   c. Consequence runs logic (enabled, chance, business logic)
   d. Consequence returns result code
   e. If success, increment rate limit counter
   f. If OK_STOP or ERROR_STOP, stop chain
9. Side effects execute inside consequences (movement, PDA, markers)

---

## Cause Patterns

### Unified Cause Producer

```
X-Ray Callback -> Producer -> Predicate(trace, ...args) -> { cause, ...data } | nil -> xbus
```

| Source Callback | Causes |
|-----------------|--------|
| `squad_on_npc_death` | massacre, basekill, squadkill, elite, elitekill |
| `squad_on_enter_smart` | stash, area, elitespot (planned), areas (planned) |
| `squad_on_leave_smart` | stash, area, elitespot (planned), areas (planned) |
| `x_npc_medkit_use` | wounded (NPC) |
| `actor_on_item_use` | wounded (player) |
| `actor_on_item_take` | harvest (player) |
| `npc_on_item_take` | harvest (NPC) |

**Predicate signature:** `function(trace, ...callback_args) -> { cause = CAUSE.X, ...payload } | nil`

**Producer responsibilities:**
- Subscribe to X-Ray callbacks
- Create trace, wrap predicate in observe()
- If predicate returns data with `.cause`, publish to xbus
- Increment rate counter

**Predicate responsibilities:**
- Return `{ cause = CAUSE.X, ...payload }` or `nil`
- No inner observe() - producer already wraps
- No event_publish() - producer handles publishing
- No trace creation - producer provides trace

### Synthetic Callbacks

Some causes need events the engine doesn't provide natively. `xevent` hooks intercept game functions and emit synthetic callbacks with `x_` prefix (e.g., `xr_eat_medkit.consume_medkit` -> `x_npc_medkit_use` for NPC wounded). Why not `actor_on_hit_callback`? It fires 50+/sec during continuous damage. Item use fires ~0.6/min.

---

## Tracker (ap_alife_tracker)

Central state manager for all persistent AP data. Owns:

| Data | API | Persistence |
|------|-----|-------------|
| Kill counts per NPC | `register_kill()`, `get_kill_count()`, `projected_kill_count()` | save/load |
| Elite registry (id -> level, kills, name, level_id) | `update_elite()`, `get_elite()`, `get_elite_level()`, `get_elites_on_level()`, `is_elite()`, `compute_level()` | save/load |
| Scripted squads (movement ownership) | `script_squad()`, `script_actor_target()`, `unscript_squad()` | save/load |
| Conquered smarts | `conquer_smart()`, `get_conquered_smarts()` | save/load (two-phase) |
| Arrival handlers | `register_arrival_handler()` | re-registered on_game_start |

**Registration order matters:** tracker registers its squad_on_npc_death handler on `on_game_start`. Producer registers on `actor_on_first_update` (later). Tracker always processes kills first. Causes read tracker data synchronously -- `projected_kill_count()` includes the current death before `register_kill()` commits it.

---

## Tracing

AP uses `observe()` wrappers (from `ap_debug`) to build hierarchical execution traces. Each trace has a monotonic `tid` -- grep `tid=N` to follow a full chain. Producer creates the trace for cause predicates; consumer passes it through via `event_data._trace`. Predicates and handlers extend the trace by calling `observe(trace, ACTION.X, fn)`. When `log_level < DEBUG`, observe() is a straight pass-through with zero overhead.

---

## Rate Limiting

```
dispatch() gate order:
  1. distributor_check (callback budget per type)
  2. _resolve_on_map (level comparison)
  3. _ratio_gate (Bresenham admission, on-map vs off-map)
  4. acquire_cause_lock (class-isolated cooldown)
  5. per-cause rate limit (sliding window)
  6. predicate evaluation
```

### Distributor (gate 1)

Limits raw X-Ray callbacks by type. Runs first -- cheapest rejection point.

| Setting | Default | What it does |
|---------|---------|--------------|
| `distributor_max_xray_events` | 10 | Max events per callback type per window |
| `distributor_window_sec` | 60 | Window duration |

Radiant callbacks (enter/leave smart) get 2x budget.

### Going Speed (gate 2+3+4, Bresenham admission control + class-isolated cooldown)

X-Ray processes A-Life at two speeds: per-frame on the player's level (`current_level_going_speed`), round-robin on distant levels (`going_speed`). With ~30 maps, off-map events outnumber on-map ~50:1. Without intervention, the player's map is starved.

AlifePlus enforces a configurable ratio using **Bresenham-derived admission control**. The core technique is the same integer cross-multiplication that Bresenham's line algorithm uses to decide pixel steps without floating-point division. Two counters (`_on_count`, `_off_count`) are the X and Y axes; the gs weight is the slope. The throttled class passes only when its count satisfies:

```
gs > 0 (favor on-map):  _off_count * gs <= (10 - gs) * _on_count
gs < 0 (favor off-map): _on_count * |gs| <= (10 - |gs|) * _off_count
gs = 0:                 gate disabled, all pass
```

At gs=8: for every 4 on-map causes, 1 off-map is allowed (ratio 4:1). Deterministic, self-adjusting. Counters reset to zero when their sum reaches 32768 to prevent unbounded growth.

After the ratio gate, events acquire a class-isolated cooldown lock. On-map and off-map have separate lock timestamps with the same duration (`global_cause_lock_sec`). The two classes never compete.

| Setting | Default | Range | What it does |
|---------|---------|-------|--------------|
| `going_speed` | 8 | -10 to +10 | How many out of 10 cause slots go to the favored class |
| `global_cause_lock_sec` | 30s | 1-300 | Cooldown after a cause fires. Separate per class. |

Level extraction: `xlevel.get_level_id(first_arg)` for squad/NPC callbacks. Actor callbacks always on-map. Unknown -> gate bypassed, off-map lock used.

### Per-Key Sliding Window (gate 5)

Uses `xttltable.create_ttl_counter`. Counts events in time window, blocks if count >= max.

| Scope | Owner | Settings | What it does |
|-------|-------|----------|--------------|
| Per-cause | Producer | `cause_window_max_events`, `cause_window_sec` | Budget per cause key |
| Per-consequence | Consumer | `consequence_window_max_events`, `consequence_window_sec` | Budget per consequence key |

Additionally, `ap_alife_behavior` uses a 1s keyed lock (`HIT_MODIFIER_LOCK_SEC`) on `npc_on_before_hit` to throttle hit modifier applications. Predicates never check rate limits -- orchestrators enforce all limiting.

---

## Result Codes

Pattern: `OUTCOME_PROPAGATION`. See `conventions.md` for full rules.

| Code | Meaning | Chain |
|------|---------|-------|
| OK_STOP | Success (exclusive) | stops |
| OK_NEXT | Success (non-exclusive) | continues |
| RULES_NEXT | Conditions not met | continues |
| CHANCE_NEXT | Chance roll failed | continues |
| DISABLED_NEXT | Enabled gate off | continues |
| THROTTLED_NEXT | Rate limited | continues |
| ERROR_STOP | Handler failure | stops |

---

## Causes

| Cause | Type | Callbacks | Trigger |
|-------|------|-----------|---------|
| MASSACRE | reactive | squad_on_npc_death | stalker victim; squad at smart (not base); death count at that smart >= threshold within decay window |
| SQUADKILL | reactive | squad_on_npc_death | squad:npc_count() <= 1 (last member dying); not at base |
| BASEKILL | reactive | squad_on_npc_death | stalker victim; squad at base; death count at base >= threshold within decay window; smart has faction |
| ELITE | reactive | squad_on_npc_death | killer NPC's projected_kill_count reaches new elite level (level > current_level) |
| ELITEKILL | reactive | squad_on_npc_death | victim NPC is tracked elite (is_elite); cooldown per victim_id |
| WOUNDED | reactive | x_npc_medkit_use, actor_on_item_use | NPC uses medkit (xevent hook on xr_eat_medkit) or actor uses healing item; not at base |
| HARVEST | reactive | actor_on_item_take, npc_on_item_take | picked up item passes IsArtefact() |
| STASH | radiant | squad_on_enter/leave_smart | xstash.find_stashes within RADIUS_NEAR_MAX finds a stash (no community filter, consequences filter) |
| AREA | radiant | squad_on_enter/leave_smart | evaluating squad is community_stalker; claimable smart (empty + not base) within RADIUS_DISTANT_MAX |
| ELITESPOT (planned) | radiant | squad_on_enter/leave_smart | elite NPC within RADIANT_SIGHT_RADIUS; evaluating squad has no elite member |
| AREAS (planned) | radiant | squad_on_enter/leave_smart | evaluating squad at own-faction conquered smart; backup squad present at same smart; another own-faction conquered smart exists on same level |

---

## Consequences

| Cause | Consequence | p | Conditions | Effect |
|-------|-------------|---|------------|--------|
| MASSACRE | massacre_scavenge | 10 | community_scavenger squads within 50-500m | find_squads_observed(community_scavenger), script_squad to massacre smart |
| MASSACRE | massacre_investigate | 20 | victim's faction squads within 50-500m | find_squads_observed(event_data.faction), script_squad to massacre smart |
| SQUADKILL | squadkill_revenge | 15 | victim_faction is community_stalker; killer exists (squad or actor) | find_squads_observed(victim_faction, 50-500m), chase_start to killer_squad or actor |
| SQUADKILL | squadkill_flee | 25 | victim_faction is community_stalker; victim_faction base on level | find_squads_observed(victim_faction, 50-500m), script_squad to nearest victim_faction base |
| BASEKILL | basekill_support | 15 | base's factions squads within 50-500m | find_squads_observed(event_data.factions), script_squad to base smart |
| BASEKILL | basekill_flee | 25 | another base_faction base on level (exclude attacked base); base_factions squads within 0-50m | find_squads_observed(event_data.factions, 0-50m), script_squad to distant base |
| WOUNDED | wounded_hunt | 15 | community_predator squads within 50-500m; nearest smart to wounded position exists | find_squads_observed(community_predator), script_squad to wounded NPC's nearest smart |
| WOUNDED | wounded_help | 25 | wounded NPC's faction resolved (xcreature.community or character_community); same-faction squads within 50-500m; nearest smart exists | find_squads_observed(wounded_faction), script_squad to wounded NPC's nearest smart |
| ELITE | elite_promote | 15 | killer_id exists; if new elite, max_elites cap not reached | update_elite(killer_id), marker "Elite L{level}: {name}", PDA |
| ELITEKILL | elitekill_bounty | 0 | none | calculate reward (base * elite_level +/-10%, capped), give_money to killer (db.actor or xobject.go), PDA |
| ELITEKILL | elitekill_targeted | 10 | killer_id exists; other elites on same level (get_elites_on_level, exclude victim + killer); trimmed to max_squads | chase_start hunters to killer_squad or actor |
| HARVEST | harvest_hunt | 15 | community_harvest squads within 50-500m; taker resolves (actor or squad) | find_squads_observed(community_harvest), chase_start to taker_squad or actor |
| STASH | stash_loot | 10 | evaluating squad is community_stalker | script_squad to stash's nearest smart, on_arrive: loot_stash_to_npc |
| STASH | stash_ambush | 20 | evaluating squad is community_ambusher | script_squad to stash's nearest smart |
| STASH | stash_fill | 30 | evaluating squad is community_stalker | script_squad to stash's nearest smart, on_arrive: fill_stash with random items |
| AREA | area_conquer | 20 | none (cause handles community + claimable filtering) | script_squad to target smart, conquer_smart(smart.id, squad.player_id), PDA |
| ELITESPOT (planned) | elitespot_flee | - | evaluating squad is enemy of elite's faction | script_squad to nearest base |
| ELITESPOT (planned) | elitespot_follow | - | evaluating squad is ally of elite's faction | chase_start to elite's squad |
| AREAS (planned) | areas_reinforce | - | none (cause handles all filtering) | script_squad to target smart |

---

## Squad Lifecycle

Consequences move squads via `scripted_target`. Without lifecycle management, squads get permanently scripted and drain the Zone of free-roaming squads.

`scripted_target` persists across save/load (src: `sim_squad_scripted.script` STATE_Write:940 / STATE_Read:984).

### Scripted Squads

`scripted_squads` table in ap_alife_tracker is the ownership registry. Every squad AP moves is tracked here.

```
Consequence -> ap_alife_tracker -> xsquad

script_squad(squad, smart, opts)  -> register in scripted_squads -> control_squad (set scripted_target)
script_actor_target(squad)        -> register as "actor" target  -> target_actor (rush, no arrival detection)
unscript_squad(squad_id)          -> remove from scripted_squads -> release_squad (clear scripted_target)
```

### Entry Lifecycle

Each entry has a TTL of 7200s (`SCRIPTED_SQUAD_TTL`). Stale entries (squad gone or TTL expired) are cleaned before every write. Entries carry an optional `on_arrive` handler key (string, serializable) and `on_arrive_args` (table, serializable). Functions are NOT stored -- they're re-registered on game_start via `register_arrival_handler`.

### Arrival Detection

`_check_arrivals` polls every 30s. Smart targets: check `xsmart.is_arrived`, dispatch handler if present, then unscript. Actor targets have no arrival detection (engine handles pursuit natively).

### On-Arrive Handler Registry

`_arrival_handlers` is a table of `{ [key] = function(squad, args) }`. Consequences call `register_arrival_handler(key, fn)` during on_game_start. The handler key is stored in `scripted_squads[id].on_arrive` (a string). On arrival, the tracker looks up the function by key and calls it with the stored args.

Handlers re-register on every on_game_start. Keys + args in scripted_squads persist across save/load. Functions don't persist -- they're re-registered.

| Handler Key | Registered By | On Arrival |
|-------------|---------------|------------|
| stash_loot | stash_loot | loot_stash_to_npc, PDA with item names |
| stash_fill | stash_fill | fill_stash with random items |
| squadkill_chase | squadkill_revenge | re-script to killer's current smart (max N re-scripts) |
| harvest_chase | harvest_hunt | re-script to taker's current smart (max N re-scripts) |
| elitekill_chase | elitekill_targeted | re-script to killer's current smart (max N re-scripts) |
| elitespot_chase | elitespot_follow (planned) | re-script to elite's current smart (max N re-scripts) |

Chase handlers use `chase_start` / `chase_register` from ap_utils. On each arrival, the handler re-scripts the squad to the target's new position. After max re-scripts, the squad is unscripted.

### Cleanup

| Function | Trigger | Scope |
|----------|---------|-------|
| `_cleanup_stale` | Before every write to scripted_squads | Remove entries where squad entity is gone or TTL expired |
| `_process_arrival` | Every 30s | Arrival detection + handler dispatch only |
| `_periodic_sync` | Every 600s (MCM-togglable) | GC all entity tables (elites, kills, conquered_smarts, scripted_squads) |

### Save/Load

- **Save:** scripted_squads persisted (all values are strings, numbers, tables -- serializable)
- **Load:** scripted_squads restored, `tracked_at` reset to `os.clock()` for all entries
- **No release-all on load.** Squads stay scripted. Handler functions re-registered on on_game_start.

---

## A-Life Behavior (ap_alife_behavior)

Runtime combat modifiers for elite NPCs. Hooks into `npc_on_before_hit`. 10-level elite system: `level = min(10, floor(kills / elite_kills_per_level))`.

### Elite Buffs

Applied on hit via `_on_before_hit`. Gated by `acquire_lock` (1s throttle). All buffs scale by elite level.

| Buff | Mechanism | Scaling | MCM Toggle |
|------|-----------|---------|------------|
| Grenades | `xobject.create_item` on hit if none in inventory | count = level | `elite_grenade_restock_enabled` |
| AP ammo | Best k_ap ammo for equipped weapon | boxes = ceil(level / 2) | `elite_ap_ammo_enabled` |
| Rank boost | `go:set_character_rank(threshold)` | 10-level threshold table | `rank_boost_enabled` |

### Rank Threshold Table

| Level | Engine Rank | Engine Label |
|-------|-------------|-------------|
| 1-2 | 10000 | experienced |
| 3-4 | 20000 | professional |
| 5-6 | 25000 | veteran |
| 7-8 | 35000 | master |
| 9-10 | 50000 | legend |

Rank boost is one-way (never reduces). Engine benefits: 20% tighter dispersion at high rank, fewer weapon jams.

AP ammo detection is weapon-pack agnostic: picks the highest `k_ap` ammo for the equipped weapon, cached per weapon section.

---

## Conquered Area

Persistent area ownership via respawn mutation. When a squad conquers a smart, `faction_controlled` and `respawn_params` are set at runtime, making the engine's `try_respawn` spawn the conqueror's faction.

### Flow

Three layers: xsmart (stateless primitives), ap_alife_tracker (lifecycle + persistence), consequence (trigger).

```
CONSEQUENCE                    AP_ALIFE_TRACKER                 XSMART
area_conquer ->               conquer_smart(smart_id, fac) ->  set_faction_controlled(smart, fac, n)
                                 conquered_smarts[id] = {         smart.faction_controlled = ALL_FC
                                   faction, conquered_at          smart.respawn_params[fc_*] = entries
                                 }                                smart.faction = fac
```

### Runtime Mutation (xsmart)

`set_faction_controlled(smart, faction, spawn_num)` mirrors engine `read_params` (smart_terrain.script:246-267). Injects `faction_controlled` entries for all 12 factions (any faction taking the smart gets matching respawns), preserves engine's `already_spawned` counts, and sets `smart.faction` to conqueror. Original non-fc entries become dormant (fc check at try_respawn blocks them).

`clear_faction_controlled(smart)` removes all 12 entries and reverts `smart.faction` to `smart.default_faction`.

### FIFO Eviction

`conquered_smarts` tracks all active conquests. When count reaches `area_conquest_max_smarts` (MCM, default 50), the entry with the lowest `conquered_at` is evicted. Uses a monotonic sequence counter (`_conquest_seq`) instead of wall clock -- survives save/load without timestamp normalization.

Re-conquest by same faction refreshes the sequence number (LRU behavior). Re-conquest by different faction overwrites the entry without eviction (count stays the same).

### Save/Load (Two-Phase)

```
1. save_state:     conquered_smarts + _conquest_seq persisted
2. load_state:     _conquered_pending = saved data, _conquest_seq restored
                   conquered_smarts = {} (cleared for fresh session)
3. STATE_Read:     engine rebuilds smart data from ltx (our mutations lost)
4. on_game_load:   iterate _conquered_pending, re-apply set_faction_controlled
                   entities exist, mutations applied before first update tick
5. periodic_sync:  stale smart IDs (entity destroyed) removed from table
```

### ZCP Compatibility

ZCP's `try_respawn` (line 1672) uses identical `faction_controlled` check as vanilla. Only adds population factor gates. Runtime mutations to `faction_controlled` / `respawn_params` are fully compatible.
