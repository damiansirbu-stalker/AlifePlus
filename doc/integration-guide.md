# AlifePlus Integration Guide

The public API for foreign mods is `ap_api`. It covers ownership coordination, reading scripted-squad state, releasing squads, registering causes and consequences, and resolving localized text. Anything outside `ap_api` is internal and not part of the contract.

Requires: AlifePlus loaded. See `architecture.md` for internals.

---

## API Reference

### Ownership

```lua
ap_api.register_owner(name, fn)
-- Register a squad ownership filter. Squads where fn(squad) returns true
-- are excluded from AP routing. Replaces existing filter on name match.
-- @param name      string  Owner name (e.g. "warfare")
-- @param fn        function(squad) -> boolean
-- @return boolean  true on success, false if args invalid

ap_api.unregister_owner(name)
-- Remove a squad ownership filter by name. No-op if not registered.
-- @return boolean  true if removed, false if not found

ap_api.get_owner(squad)
-- Query squad ownership.
-- @param squad     userdata  Squad server object
-- @return string|nil          Owner name, or nil if unowned
```

### Scripted-squad state

```lua
ap_api.get_scripted_squads()
-- Returns a shallow-copied snapshot of the AP-scripted squads table, keyed by squad id.
-- Outer table and each entry are fresh per call; mutation does not affect broker state.
-- Each entry carries:
--   consequence       string   the squad's current consequence (e.g. CONSEQUENCE.MASSACRE_INVESTIGATE)
--   action            string   the player-facing action (e.g. ACTION.MASSACRE_INVESTIGATE). Pass to ap_api.get_text.
--   scripted_target   string   smart name, or "actor" for actor-pursuit
--   target_smart_id   number   smart id (nil for actor-pursuit)
--   tracked_at        number   game-time the script started
--   pre_release_gulag number   seconds the broker holds the squad after arrival
--   on_arrive         string   internal arrival-handler dispatch key
--   on_arrive_args    table    args passed to the arrival handler
--   arrived           boolean  set true after arrival (optional)
--   release_at        number   game-time of pending release (optional)
-- @return table
-- Prefer get_scripted_squad(id) for single-entry lookups (O(1), no allocation).

ap_api.get_scripted_squad(squad_id)
-- Returns a shallow copy of the entry for one squad id, or nil.
-- Fresh per call; mutation does not affect broker state.
-- @param squad_id  number
-- @return table|nil

ap_api.release_squad(squad_id)
-- Release a squad from AP scripted control. Stops the broker's reassert loop
-- and clears scripted_target.
-- @param squad_id  number
-- @return boolean  false if AP wasn't currently scripting the squad

ap_api.script_squad(squad, smart, opts)
-- Script a squad to a smart terrain with full AP lifecycle (PDA marker, TTL,
-- pre-release hold, target reassertion).
-- @param squad     userdata
-- @param smart     userdata
-- @param opts      table { consequence, action, rush?, on_arrive?, on_arrive_args?, pre_release_gulag? }
--   consequence (required) - CONSEQUENCE.X enum value (which consequence dispatched this squad)
--   action      (required) - ACTION.X enum value (player-facing label; see ap_api.get_text)
--   rush                   - run + danger anim when online
--   on_arrive              - dispatch key, only set when a consequence with
--                            this name registered an arrival handler
--   on_arrive_args         - passed to the arrival handler
--   pre_release_gulag      - seconds held after arrival (default from const)
-- @return table { code, id, dst, dst_id }

ap_api.script_actor_target(squad, opts)
-- Script a squad to pursue the player. No arrival detection (player moves).
-- @param squad     userdata
-- @param opts      table { consequence, action }
--   consequence (required) - CONSEQUENCE.X enum value
--   action      (required) - ACTION.X enum value
-- @return table { code, id, dst, dst_id }
```

### Display text

```lua
ap_api.get_text(id)
-- Resolve a CONSEQUENCE.X or ACTION.X enum value to its localized phrase.
-- Both enum sets share one localization file: ui_st_ap_consequence.xml.
--   ACTION.X      -> "action:<key>"      -> "Investigating a Massacre Site"
--   CONSEQUENCE.X -> "consequence:<key>" -> "Massacre Investigate"
-- The enum value is itself the localization id; get_text just translates.
-- @param id   string   CONSEQUENCE.X or ACTION.X enum value
-- @return string|nil   localized text, or nil for unknown / unresolved ids
```

The PDA marker line for an AP-scripted squad shows the action phrase as the
headline, the consequence caption in brackets underneath, and the subject line:

```
Investigating a Massacre Site
[Massacre Investigate]
Subject: Loner Squad (Loners)
```

### Cause and consequence registration

```lua
ap_api.get_cause(consequence)
-- The cause string a registered consequence subscribes to (1:1 mapping).
-- @param consequence  string   consequence registration name
-- @return string|nil           cause string ("cause:massacre"), or nil

ap_api.register_radiant_cause(category, predicate)
-- Register a radiant cause predicate. Auto-iterates RADIANT_CALLBACKS.
-- Predicate fires per radiant tick with one squad and returns either a
-- publishing payload, a rejection table, or nil.
-- @param category  string  Must be one of ap_core_const.CAUSE_CATEGORY values
-- @param predicate function(squad) -> table|nil
-- @return boolean  true if all registrations succeeded, false on category invalid or producer rejection

ap_api.register_reactive_cause(callback, category, predicate)
-- Register a reactive cause predicate for one engine callback.
-- Predicate signature matches the engine callback's args.
-- @param callback   string  CALLBACK enum value
-- @param category   string  Must be one of ap_core_const.CAUSE_CATEGORY values
-- @param predicate  function(...) -> table|nil
-- @return boolean  true on success, false on category invalid or producer rejection

ap_api.register_consequence(name, opts, handler)
-- Register a consequence handler. Strict: refuses duplicate names.
-- Call unregister_consequence(name) first to replace.
-- @param name     string  Consequence registration name (also the arrival dispatch key)
-- @param opts     table { cause, condition?, on_arrive_fn? }
--   cause         (required) - cause string the handler subscribes to
--   condition     (optional) - function() -> bool; false to skip dispatch entirely
--   on_arrive_fn  (optional) - function(squad, on_arrive_args) called when a squad
--                              scripted by this consequence arrives at its smart
-- @param handler  function(event_data) -> { code = RESULT.X, ... }
-- @return boolean  true on success, false if already registered or args invalid

ap_api.unregister_consequence(name)
-- Remove a consequence handler by name. Removes from event registry, clears
-- cause-of mapping, and unhooks any arrival handler under the same name.
-- @param name  string  Consequence registration name
-- @return boolean  true if removed, false if not registered
```

### Predicate payload contract

A predicate that publishes a cause must return a table with at least:

```lua
{
    cause     = "cause:<modname>:<event>",  -- routing key, must be unique
    squad_id  = squad.id,                   -- for AP debug logging
    level_id  = squad.level_id,             -- for AP debug logging
    -- ...any other fields your consequence needs
}
```

A predicate that rejects returns either nil, or a structured rejection:

```lua
{ code = RESULT.FAILED_RULES, reason = REASON.WRONG_ALIGNMENT }
```

### Categories

`category` must be one of:

```lua
ap_core_const.CAUSE_CATEGORY.REACTIONS      -- triggered by world events (NPC died, item used)
ap_core_const.CAUSE_CATEGORY.NEEDS          -- triggered by squad internal state (eat, sleep)
ap_core_const.CAUSE_CATEGORY.INSTINCTS      -- triggered by stimulus (scent, sense of danger)
ap_core_const.CAUSE_CATEGORY.OPPORTUNITIES  -- triggered by environmental state (empty smart, abandoned stash)
```

Foreign causes share AP's per-category MCM caps and stats grouping. `register_radiant_cause` and `register_reactive_cause` reject unknown categories.

### Consequences and actions

Every call into `script_squad` / `script_actor_target` MUST pass both `consequence = CONSEQUENCE.X` and `action = ACTION.X`. The broker stores both on the record:

- `consequence` — short caption. Routing key for arrival dispatch and the bracketed sub-label on the marker. Example: `"consequence:massacre_investigate"` -> "Massacre Investigate".
- `action` — full action phrase. Headline on the marker. Example: `"action:massacre_investigate"` -> "Investigating a Massacre Site".

Both enum values are themselves localization ids. They share one file per locale: `gamedata/configs/text/eng/ui_st_ap_consequence.xml` and `gamedata/configs/text/rus/ui_st_ap_consequence.xml`. The Russian file is windows-1251; edit it via `xmlstarlet` only.

`ap_api.get_text(id)` is `game.translate_string(id)` and works for both consequence and action ids.

For foreign mods adding new consequences:

1. Pick keys, define your enum constants:
   ```lua
   local CONSEQUENCE_AMBUSH_SETUP = "consequence:my_mod_ambush_setup"
   local ACTION_AMBUSH_SETUP      = "action:my_mod_ambush_setup"
   ```
2. Add matching `<string id="...">` entries in your mod's own `ui_st_<modname>.xml` (eng + rus) with both ids.
3. Pass both as `consequence = ...` and `action = ...` in `script_squad` opts.

If you only ship one of the two strings, the marker falls back to the raw enum value (e.g. `[consequence:my_mod_ambush_setup]`) for the missing side.

---

## Integration Scenarios

### Detect AlifePlus

```lua
local function has_alifeplus()
    return ap_api ~= nil
end
```

`ap_api` is the public namespace. If AP is loaded, `ap_api` is non-nil and every documented function exists.

### Coordinate squad ownership

So AP doesn't route to squads your mod owns. Warfare's pattern:

```lua
function on_game_start()
    ap_api.register_owner("warfare", function(squad)
        if not squad then return false end
        return squad.registered_with_warfare == true
    end)
end
```

`register_owner` replaces the existing filter on name match, so calling it again with the same name updates the filter. AP excludes owned squads at multiple layers (producer gate, cause predicate, find_squads, scripting).

To stop owning squads (e.g. on MCM toggle):

```lua
ap_api.unregister_owner("warfare")
```

### Display AP activity in your UI

Tooltip showing what AP is doing with a squad:

```lua
local function ap_action_for(squad_id)
    if not ap_api then return nil end
    local entry = ap_api.get_scripted_squad(squad_id)
    if not entry or not entry.action then return nil end
    return ap_api.get_text(entry.action)
end

-- Call site:
local text = ap_action_for(squad.id)
if text then
    tooltip = tooltip .. "\n" .. text
end
```

Returns localized strings: "Investigating a Massacre Site", "Heading to a Campfire to Rest", etc. nil when the squad isn't currently scripted by AP. Call fresh on every render tick; entries change as squads move between consequences.

### Take a squad back from AP

When your mod needs a specific squad immediately:

```lua
if ap_api.release_squad(squad.id) then
    -- AP was scripting it; broker's reassert loop is now stopped
end
xsquad.acquire_squad(squad, your_smart, true)
```

`release_squad` returns `false` if AP wasn't scripting the squad. Safe to call unconditionally.

### Add a new cause and consequence

Foreign mod adds its own cause + consequence. Goes through AP's full pipeline (gates, rate limits, PDA news, marker lifecycle).

```lua
local CAUSE_AMBUSH = "cause:my_mod:ambush"
local CONSEQUENCE_AMBUSH_SETUP = "MY_MOD_AMBUSH_SETUP"
local ACTION_AMBUSH_SETUP = "action:my_mod_ambush_setup"
local CATEGORY = ap_core_const.CAUSE_CATEGORY.OPPORTUNITIES
local cfg = my_mod_mcm.cfg

-- Cause: outlaws notice an empty smart, want to ambush
local function _predicate(squad)
    if not cfg.ambush_enabled then return nil end
    if not is_outlaw(squad) then return nil end
    if not ap_core_limiter.check_cause_rate_limit(CATEGORY, cfg.cause_max_opportunities) then
        return { code = ap_core_const.RESULT.FAILED_RULES,
                 reason = ap_core_const.REASON.BUDGET_EXHAUSTED }
    end
    local smart = find_empty_smart_near(squad)
    if not smart then return nil end
    ap_core_limiter.increment_cause_counter(CATEGORY)
    return {
        cause     = CAUSE_AMBUSH,
        squad_id  = squad.id,
        level_id  = squad.level_id,
        smart_id  = smart.id,
        community = squad.player_id,
    }
end

-- Consequence: send nearby outlaws to the smart
local function _on_arrive(squad, args)
    log("squad %d arrived to ambush at smart %d", squad.id, args.smart_id)
end

local function _handler(event_data)
    local smart = xobject.se(event_data.smart_id)
    if not smart then return { code = ap_core_const.RESULT.FAILED_SCAN } end
    local squads = find_outlaw_squads_near(event_data)
    for _, squad in ipairs(squads) do
        ap_api.script_squad(squad, smart, {
            consequence       = CONSEQUENCE_AMBUSH_SETUP,
            action            = ACTION_AMBUSH_SETUP,
            on_arrive         = CONSEQUENCE_AMBUSH_SETUP,
            on_arrive_args    = { smart_id = event_data.smart_id },
            rush              = true,
            pre_release_gulag = 1800,
        })
    end
    return { code = ap_core_const.RESULT.SUCCESS }
end

function on_game_start()
    ap_api.register_radiant_cause(CATEGORY, _predicate)
    ap_api.register_consequence(CONSEQUENCE_AMBUSH_SETUP, {
        cause        = CAUSE_AMBUSH,
        condition    = function() return cfg.ambush_setup_enabled end,
        on_arrive_fn = _on_arrive,
    }, _handler)
end
```

Conventions:

- Cause strings: namespace as `"cause:<modname>:<event>"` to avoid collision with AP's standard causes
- Consequence names: namespace similarly (`"MY_MOD_X"` or your own pattern)
- Pass the same string for `consequence` and `on_arrive` if you want the consequence to handle its own arrivals
- Self-gate against the per-category cause budget with `ap_core_limiter.check_cause_rate_limit` and `increment_cause_counter`; the framework does not gate this for you
- Predicate payload must include `squad_id` and `level_id` (AP debug log reads them)

### Resolve localized text outside the tooltip pattern

For any consequence or action enum value you have:

```lua
local action_text     = ap_api.get_text(ap_core_const.ACTION.MASSACRE_INVESTIGATE)
-- "Investigating a Massacre Site"

local consequence_text = ap_api.get_text(ap_core_const.CONSEQUENCE.MASSACRE_INVESTIGATE)
-- "Massacre Investigate"
```

Both enum values are themselves localization ids (`"action:massacre_investigate"`, `"consequence:massacre_investigate"`). Strings live in `gamedata/configs/text/<lang>/ui_st_ap_consequence.xml`. AlifePlus includes English and Russian; locale switches via `game.translate_string`.

---

## Notes

- `ap_api` is the contract. `ap_core_*` modules are internal; using them directly is possible but unsupported across versions.
- For an alternative integration that doesn't use `ap_api`, you can subscribe directly to AP's xbus events. Cause names are listed in `ap_core_const.CAUSE`. Listener-only mods (telemetry, journal, sound) typically use this path.

### Backward-compatibility shims

`ap_core_compat.script` keeps pre-`ap_api` call shapes alive for one mod that pre-dates the public API: the **Warfare A Life Overhaul GAMMA fork** by erepb. Specifically:

| Old call site                                                | Shim                                                      |
|--------------------------------------------------------------|-----------------------------------------------------------|
| `ap_core_broker.get_scripted_ids()[id]`                      | aliased to `get_scripted_squads`                          |
| `ap_core_broker.get_record({squad_id = id})`                 | synthesized; returns `{ consequence = entry.consequence }`|
| `ap_core_const.CONSEQUENCE_INFO[c].action_key`               | rebuilt from CONSEQUENCE+ACTION; `action_key` is the action enum value, which `game.translate_string` resolves directly |

New integrations should not rely on these. They exist for one specific external consumer; the surface may shrink without notice once that consumer migrates.

---

## Contact

Questions, integration requests, API feedback:

- Discord: **damian_sirbu**
- ModDB: **damian_sirbu**
- Email: **dami.sirbu@gmail.com**
- GitHub: **github.com/damiansirbu-stalker**
