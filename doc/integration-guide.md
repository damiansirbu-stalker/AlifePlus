# AlifePlus Integration Guide

Three integration levels, from passive listener to full pipeline participant.

Requires: xlibs 1.2.2+, AlifePlus loaded. See `architecture.md` for internals, `conventions.md` for naming.

---

## Integration Levels

| Level | What you do | What you get | Dependency |
|-------|------------|--------------|------------|
| 1. Listen | Subscribe to xbus events | Cause data (position, level, entities) | xlibs only |
| 2. Register | Register cause predicate + consequence handler | Protection, rate limiting, PDA, markers, lifecycle | ap_core_* |
| 3. Coordinate | Use tracker, conquest, or squad internals | Domain state, territory control, squad ownership | ap_core_* + ap_ext_* |

---
---

## Register: Add a Cause + Consequence

The fastest way to extend AP. Register a predicate and a handler. The framework handles gates, protection, rate limiting, tracing, PDA routing, squad lifecycle.

---

### Cause (predicate)

A cause evaluates a world-state condition and returns structured data or nil.

```lua
-- ap_ext_cause_ambush.script
local CAUSE = ap_core_const.CAUSE
local CAUSE_TYPE = ap_core_const.CAUSE_TYPE
local cfg = ap_core_mcm.cfg

local function _predicate(trace, squad)
    if not cfg.cause_ambush_enabled then return nil end
    local smart = xsmart.nearest_empty(squad, 300)
    if not smart then return nil end
    return {
        cause = CAUSE.AMBUSH,
        squad_id = squad.id,
        smart_id = smart.id,
        level_id = xlevel.get_level_id(squad),
        position = squad.position,
    }
end

function on_game_start()
    local cbs = ap_core_const.RADIANT_CALLBACKS
    for i = 1, #cbs do
        ap_core_producer.register(CAUSE.AMBUSH,
            { callback = cbs[i], cause_type = CAUSE_TYPE.RADIANT },
            _predicate)
    end
end
```

Rules:
- Return `{ cause = CAUSE.X, ...payload }` or nil
- Never call observe(), publish(), or manipulate counters
- Include `level_id` from the source entity, never the actor
- Add `CAUSE.AMBUSH = "cause:ambush"` to ap_core_const
- Add MCM `cause_ambush_enabled` to ap_core_mcm.defaults

---

### Consequence (handler)

A consequence receives a cause event and returns a result code.

```lua
-- ap_ext_consequence_ambush_setup.script
local RESULT = ap_core_const.RESULT
local REASON = ap_core_const.REASON
local CONSEQUENCE = ap_core_const.CONSEQUENCE
local CAUSE = ap_core_const.CAUSE
local cfg = ap_core_mcm.cfg

local function _handler(event_data)
    if not cfg.consequence_ambush_setup_enabled then
        return { code = RESULT.DISABLED_NEXT }
    end
    local smart = xobject.se(event_data.smart_id)
    if not smart then
        return { code = RESULT.RULES_NEXT, reason = REASON.NO_SMART }
    end

    local squads = ap_core_utils.find_squads_observed(event_data._trace, event_data.position, {
        factions = ap_ext_const.community_outlaw,
        level_id = event_data.level_id,
        max_distance = ap_core_const.RADIUS_DISTANT_MAX,
    })
    if not squads or #squads == 0 then
        return { code = RESULT.RULES_NEXT, reason = REASON.NO_SQUADS }
    end

    for i = 1, #squads do
        ap_core_squad.script_squad(squads[i], smart, {
            on_arrive = CONSEQUENCE.AMBUSH_SETUP,
        })
    end

    return { code = RESULT.OK_STOP, count = #squads }
end

function on_game_start()
    ap_core_consumer.register(CONSEQUENCE.AMBUSH_SETUP, {
        event = CAUSE.AMBUSH,
        personality = { PERSONALITY.AGGRESSION, PERSONALITY.GREED },
    }, _handler)
end
```

Rules:
- Always return `{ code = RESULT.X }`
- Gate order: enabled -> personality -> data validation -> logic -> result
- `personality` declares which faction traits gate this consequence (average of traits rolled per evaluation)
- OK_STOP = exclusive (stops chain), OK_NEXT = non-exclusive (continues)
- Use `find_squads_observed` for traced search (protections applied automatically)
- Add `CONSEQUENCE.AMBUSH_SETUP = "consequence:ambush_setup"` to ap_core_const

---

### Arrival behavior

Register an `on_arrive` function to run when the squad reaches its destination:

```lua
local function _on_arrive(squad, args)
    -- runs when squad reaches destination
end

function on_game_start()
    ap_core_consumer.register(CONSEQUENCE.AMBUSH_SETUP, {
        event = CAUSE.AMBUSH,
        personality = { PERSONALITY.AGGRESSION, PERSONALITY.GREED },
        on_arrive = _on_arrive,
    }, _handler)
end
```

Link arrival to squad: pass `on_arrive = CONSEQUENCE.AMBUSH_SETUP` in `script_squad` opts.

---

### Chase (re-script on arrival)

```lua
local function _on_arrive(squad, args)
    if not args then return end
    local target = xobject.se(args.target_id)
    if not target then return end
    args.chase_count = (args.chase_count or 0) + 1
    if args.chase_count > cfg.consequence_ambush_chase_max then return end
    local smart = xsmart.get_smart(target)
    if smart then
        ap_core_squad.script_squad(squad, smart, {
            rush = true,
            on_arrive = CONSEQUENCE.AMBUSH_CHASE,
            on_arrive_args = args,
        })
    end
end
```

---

### Checklist

1. Add `CAUSE.X` to ap_core_const CAUSE table
2. Add `CONSEQUENCE.X_VERB` to ap_core_const CONSEQUENCE table
3. Add MCM defaults to ap_core_mcm.defaults
4. Create `ap_ext_cause_x.script` with predicate, register in on_game_start
5. Create `ap_ext_consequence_x_verb.script` with handler, register in on_game_start
6. Add PDA messages to ap_ext_messages if needed
7. Add MCM UI entries to ap_core_mcm menu builder
8. Run validator: `stalker-manager.sh validate`

---
---

## Two Alife Mods Collaborating

ModA controls squads for territorial warfare. AP controls squads for emergent behavior. Neither knows the other's internals.

---

### Squad coordination

`scripted_target` is the engine's built-in coordination mechanism. When any mod sets `scripted_target` on a squad, the engine switches it from `generic_update` (simulation re-evaluation) to `specific_update` (direct A->B movement).

Both mods check `scripted_target` before claiming a squad. This is enough for basic non-interference:

```lua
-- ModA before scripting:
if se_squad.scripted_target then return end  -- AP (or anyone) has this squad
se_squad.scripted_target = smart.id

-- AP's PROTECTION gate already does this check automatically.
-- No code change needed on AP's side.
```

---

### Ownership registry (identity)

For knowing WHO controls a squad (not just that it's controlled), register an ownership filter:

```lua
-- In ModA's on_game_start:
ap_core_squad.register_owner("warfare", function(squad)
    return squad.registered_with_warfare == true
end)
```

AP already registers warfare and BAO in `ap_core_compat`:

```lua
ap_core_squad.register_owner("warfare", function(squad)
    return squad.registered_with_warfare == true
end)
ap_core_squad.register_owner("bao", function(squad)
    return squad.__lock == true
end)
```

---

### Result

| Situation | What happens |
|-----------|-------------|
| ModA scripts a squad | AP sees `scripted_target`, skips at PROTECTION gate |
| AP scripts a squad | ModA sees `scripted_target`, skips |
| ModA marks squad as owned (no scripted_target yet) | AP sees ownership filter, skips at PROTECTION gate |
| Neither owns the squad | Both can compete; first to set `scripted_target` wins |

No shared state. No direct imports. Just `scripted_target` + ownership filters.

---
---

## Listen: Subscribe to Events

Subscribe to cause events via xbus. No AP dependency - xlibs only.

```lua
function on_game_start()
    xbus.subscribe("cause:massacre", function(e)
        printf("massacre at %s: %d dead", e.level_id, e.total_deaths)
    end, "my_mod_listener")
end
```

---

### Events

| Event | Type | Key payload |
|-------|------|-------------|
| cause:massacre | reactive | position, level_id, smart_id, total_deaths, faction |
| cause:squadkill | reactive | position, level_id, squad_id, killer_id |
| cause:basekill | reactive | position, level_id, smart_id, total_deaths, faction |
| cause:elite | reactive | killer_id, kills, level |
| cause:elitekill | reactive | victim_id, killer_id, elite_level |
| cause:wounded | reactive | position, level_id, squad_id |
| cause:harvest | reactive | position, level_id, squad_id, item_section |
| cause:stash | radiant | position, level_id, squad_id, stash_id |
| cause:area | radiant | position, level_id, squad_id, smart_id |
| cause:needs | radiant | position, level_id, squad_id, need, drive |

All events include `_trace` and `cause_type` ("radiant" or "reactive").

---

### Use cases

- Faction relations mod: adjust goodwill on massacre/basekill
- Statistics tracker: count causes per level, per faction
- Sound mod: play ambient audio when nearby events fire
- Journal mod: log events to a player-readable journal

---
---

## Coordinate: Deep Integration

Direct access to AP domain systems. APIs may change between versions.

---

### Tracker (ap_ext_tracker)

| Function | Returns |
|----------|---------|
| `get_elite(entity_id)` | elite data table (level, kills, name) or nil |
| `get_elite_level(entity_id)` | integer level or 0 |
| `is_elite(entity_id)` | boolean |
| `get_elites()` | all elites table |
| `get_elites_on_level(level_id)` | elites on a specific map |
| `get_stalker_needs(squad_id)` | needs DTO timestamps |
| `projected_kill_count(killer_id, victim_id)` | kill count if this kill were registered |

---

### Smart Mutator (ap_ext_smart_mutator)

| Function | Returns |
|----------|---------|
| `conquer_smart(smart_id, faction)` | set faction ownership with FIFO eviction |

---

### Squad (ap_core_squad)

| Function | Returns |
|----------|---------|
| `script_squad(squad, smart, opts)` | script with lifecycle |
| `script_actor_target(squad)` | pursue player |
| `is_protected(squad)` | check all guards |
| `get_owner(squad)` | ownership query |
| `register_owner(name, filter_fn)` | register ownership filter |
| `add_marked_squad(squad_id, label, opts)` | PDA map marker |
| `get_scripted_ids()` | read-only scripted squads table |

`script_squad` does not check protection. Caller must verify:

```lua
if ap_core_squad.is_protected(squad) then return end
ap_core_squad.script_squad(squad, smart, opts)
```

---
---

## Contact

Questions, integration requests, API feedback:

- Discord: **damian_sirbu**
- ModDB: **damian_sirbu**
- Email: **dami.sirbu@gmail.com**
- GitHub: **github.com/damiansirbu-stalker**
