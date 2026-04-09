# AlifePlus Integration Guide

How to build on AlifePlus. Three integration levels, from passive listener to full pipeline participant.

Requires: xlibs 1.2.2+, AlifePlus loaded.

See `architecture.md` for system internals. See `conventions.md` for naming rules.

---

## Integration Levels

| Level | What you do | What you get | AP dependency |
|-------|------------|--------------|---------------|
| 1. Listen | Subscribe to xbus events | Cause data (position, level, entities) | xlibs only |
| 2. Register | Register cause predicate + consequence handler | Throttling, protection, rate limiting, PDA, markers, lifecycle | Core API |
| 3. Coordinate | Use tracker, conquest, or squad internals | Domain state, territory control, squad ownership | Core + Ext API |

---

## Level 1: Listen to Events

Subscribe to cause events via xbus. No AP code dependency - xlibs only.

```lua
-- my_mod.script
function on_game_start()
    xbus.subscribe("cause:massacre", function(event_data)
        -- event_data.position   : vector, where it happened
        -- event_data.level_id   : number, map ID
        -- event_data.squad_id   : number, triggering squad
        -- event_data.cause_type : "radiant" or "reactive"
        printf("massacre at level %d", event_data.level_id)
    end, "my_mod_massacre_listener")
end
```

### Available events

| Event | Type | Key payload fields |
|-------|------|--------------------|
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

All events include `_trace` (trace context) and `cause_type` ("radiant" or "reactive").

### Use cases

- **Faction relations mod:** adjust goodwill on massacre/basekill events.
- **Statistics tracker:** count causes per level, per faction.
- **Sound mod:** play ambient audio when nearby events fire.
- **Journal mod:** log events to a player-readable journal.

---

## Level 2: Register Causes and Consequences

Register your logic with AP's pipeline. You get throttling, protection, rate limiting, tracing, and PDA routing for free.

### Writing a cause

A cause is a predicate that evaluates on an engine callback and returns structured data or nil.

```lua
-- ap_ext_cause_myevent.script
local CAUSE = ap_core_const.CAUSE
local CALLBACK = ap_core_const.CALLBACK
local CAUSE_TYPE = ap_core_const.CAUSE_TYPE
local cfg = ap_core_mcm.cfg

local function _predicate(trace, squad)
    if not cfg.cause_myevent_enabled then return nil end
    -- your world-state check here
    if not my_condition(squad) then return nil end
    return {
        cause = CAUSE.MYEVENT,
        squad_id = squad.id,
        level_id = xlevel.get_level_id(squad),
        position = squad.position,
    }
end

function on_game_start()
    -- Radiant: register on all radiant callbacks
    local cbs = ap_core_const.RADIANT_CALLBACKS
    for i = 1, #cbs do
        ap_core_producer.register(CAUSE.MYEVENT,
            { callback = cbs[i], cause_type = CAUSE_TYPE.RADIANT },
            _predicate)
    end

    -- OR reactive: register on a specific callback
    -- ap_core_producer.register(CAUSE.MYEVENT,
    --     { callback = CALLBACK.SQUAD_ON_NPC_DEATH, cause_type = CAUSE_TYPE.REACTIVE },
    --     _predicate)
end
```

**Rules:**
- Predicate returns `{ cause = CAUSE.X, ...payload }` or nil. Nothing else.
- Never call observe(), publish(), or manipulate counters - producer handles all of it.
- Include `level_id` from the event source entity, never from the actor.
- Add `CAUSE.MYEVENT = "cause:myevent"` to ap_core_const.
- Add MCM `cause_myevent_enabled` to ap_core_mcm.defaults.

### Writing a consequence

A consequence handler receives event data and returns a result code.

```lua
-- ap_ext_consequence_myevent_respond.script
local RESULT = ap_core_const.RESULT
local REASON = ap_core_const.REASON
local CONSEQUENCE = ap_core_const.CONSEQUENCE
local CAUSE = ap_core_const.CAUSE
local cfg = ap_core_mcm.cfg

local function _handler(event_data)
    if not cfg.consequence_myevent_respond_enabled then
        return { code = RESULT.DISABLED_NEXT }
    end
    if not xmath.chance(cfg.consequence_myevent_respond_chance) then
        return { code = RESULT.CHANCE_NEXT }
    end

    local trace = event_data._trace
    local position = event_data.position
    local level_id = event_data.level_id

    -- Find nearby squads
    local squads = ap_core_utils.find_squads_observed(trace, position, {
        factions = ap_ext_const.community_stalker,
        level_id = level_id,
        min_distance = ap_core_const.RADIUS_CLOSE_MAX,
        max_distance = ap_core_const.RADIUS_DISTANT_MAX,
    })
    if not squads or #squads == 0 then
        return { code = RESULT.RULES_NEXT, reason = REASON.NO_SQUADS }
    end

    -- Script squads to event location
    local smart = xobject.se(event_data.smart_id)
    if not smart then
        return { code = RESULT.RULES_NEXT, reason = REASON.NO_SMART }
    end

    for i = 1, #squads do
        local move = ap_core_squad.script_squad(squads[i], smart, {
            rush = cfg.consequence_myevent_respond_rush,
        })
        if move.code == RESULT.OK_NEXT then
            ap_core_squad.add_marked_squad(squads[i].id, CONSEQUENCE.MYEVENT_RESPOND)
        end
    end

    -- PDA
    ap_core_debug.observe(trace, ap_core_const.ACTION.SEND_PDA, function()
        return ap_core_utils.send_pda(
            cfg.consequence_myevent_respond_pda_chance,
            "myevent_respond",
            { location = ap_core_utils.get_location_description(nil, position, level_id) }
        )
    end)

    return { code = RESULT.OK_STOP, count = #squads }
end

function on_game_start()
    ap_core_consumer.register(CONSEQUENCE.MYEVENT_RESPOND, {
        event = CAUSE.MYEVENT,
        priority = 20,
    }, _handler)
end
```

**Rules:**
- Always return `{ code = RESULT.X }`. Consumer checks `.code` for chain propagation.
- Gate order: enabled -> chance -> data validation -> logic -> result.
- OK_STOP = exclusive (stops chain). OK_NEXT = non-exclusive (continues).
- Use `find_squads_observed` for traced squad search (protections applied automatically).
- Add `CONSEQUENCE.MYEVENT_RESPOND = "consequence:myevent_respond"` to ap_core_const.
- Add MCM defaults for enabled, chance, rush, pda_chance.

### Adding arrival behavior

Register an on_arrive function to execute logic when the squad reaches its destination.

```lua
local function _on_arrive(squad, args)
    -- squad has arrived at its destination
    -- do inventory ops, state changes, etc.
end

function on_game_start()
    ap_core_consumer.register(CONSEQUENCE.MYEVENT_RESPOND, {
        event = CAUSE.MYEVENT,
        priority = 20,
        on_arrive = _on_arrive,  -- registered automatically
    }, _handler)
end
```

The consequence name IS the arrival handler key. When calling `script_squad`, pass `on_arrive = CONSEQUENCE.MYEVENT_RESPOND` in opts to link the arrival to this handler.

```lua
ap_core_squad.script_squad(squad, smart, {
    rush = true,
    on_arrive = CONSEQUENCE.MYEVENT_RESPOND,
    on_arrive_args = { some_data = "value" },
})
```

### Adding a chase consequence

Chase consequences re-script the squad on each arrival until the target is lost or max chases reached.

```lua
local function _on_arrive(squad, args)
    if not args then return end
    local target = xobject.se(args.target_id)
    if not target then
        -- target gone, send PDA "lost trail"
        return
    end
    -- check: same level? not co-located? under max chases?
    args.chase_count = (args.chase_count or 0) + 1
    if args.chase_count > cfg.consequence_myevent_chase_max_chases then
        return  -- give up
    end
    -- re-script to target's current position
    local smart = xsmart.get_smart(target)
    if smart then
        ap_core_squad.script_squad(squad, smart, {
            rush = true,
            on_arrive = CONSEQUENCE.MYEVENT_CHASE,
            on_arrive_args = args,
        })
    end
end
```

### Integrating existing logic

Example: warfare mod routing through AP.

```lua
-- Instead of directly scripting squads:
-- squad.scripted_target = smart.id  -- DON'T

-- Register as an ownership source:
ap_core_squad.register_owner("my_warfare", function(squad)
    return squad.my_warfare_flag == true
end)

-- OR register warfare events as AP causes/consequences:
ap_core_producer.register(CAUSE.WARFARE_CAPTURE, {
    callback = CALLBACK.SQUAD_ON_UPDATE,
    cause_type = CAUSE_TYPE.RADIANT,
}, warfare_capture_predicate)
```

---

## Level 3: Coordinate

Deep integration with AP domain systems. Contact the author before using these - APIs may change.

### Tracker (ap_ext_tracker)

- `get_elite(entity_id)` - elite data table (level, kills, name) or nil
- `get_elite_level(entity_id)` - integer level or 0
- `is_elite(entity_id)` - boolean
- `get_elites()` - all elites table
- `get_elites_on_level(level_id)` - elites on a specific map
- `get_stalker_needs(squad_id)` - needs DTO timestamps (last_hunger_at, etc.)
- `projected_kill_count(killer_id, victim_id)` - kill count if this kill were registered

### Smart Mutator (ap_ext_smart_mutator)

- `conquer_smart(smart_id, faction)` - set faction ownership with FIFO eviction

### Squad (ap_core_squad)

- `script_squad(squad, smart, opts)` - script with lifecycle
- `script_actor_target(squad)` - pursue player
- `is_protected(squad)` - check all guards
- `get_owner(squad)` - ownership query
- `register_owner(name, filter_fn)` - register ownership filter
- `add_marked_squad(squad_id, label, opts)` - PDA map marker
- `get_scripted_ids()` - read-only scripted squads table

### Important: script_squad has no inline protection

`script_squad` does not check protection. It assumes the caller already verified the squad is safe to script. Protection happens upstream:

- Producer gates (radiant pipeline)
- Cause-level is_protected checks (reactive pipeline)
- find_squads exclusion filters (consequence level)

If you call script_squad directly, you must verify protection yourself:

```lua
local protected = ap_core_squad.is_protected(squad)
if protected then return end
ap_core_squad.script_squad(squad, smart, opts)
```

---

## Checklist: Adding a New Cause + Consequence

1. Add `CAUSE.X` to ap_core_const CAUSE table
2. Add `CONSEQUENCE.X_VERB` to ap_core_const CONSEQUENCE table
3. Add MCM defaults to ap_core_mcm.defaults
4. Create `ap_ext_cause_x.script` with predicate, register in on_game_start
5. Create `ap_ext_consequence_x_verb.script` with handler, register in on_game_start
6. Add PDA messages to ap_ext_messages if needed
7. Add MCM UI entries to ap_core_mcm menu builder
8. Run validator: `stalker-manager.sh validate`

---

## Contact

Questions, integration requests, API feedback:

- Discord: **damian_sirbu**
- ModDB: **damian_sirbu**
- Email: **dami.sirbu@gmail.com**
- GitHub: **github.com/damiansirbu-stalker**
