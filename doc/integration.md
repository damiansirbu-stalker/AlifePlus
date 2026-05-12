# AlifePlus Integration

This file is for mods that want to react to AlifePlus, drive squads through it, or add their own causes and consequences. Everything you call goes through *ap_api*. *ap_core_\** modules are internal — using them directly is not supported across versions.

> [!IMPORTANT]
> Always feature-detect. *ap_api* may exist but lack a specific function; check the function before calling it. Every example below follows this convention.

---

## API surface

| Function | Params | Returns | Notes |
|---|---|---|---|
| *register_owner* | *name*: string; *fn*: function(squad) → bool | bool | Replaces existing filter on name match. |
| *unregister_owner* | *name*: string | bool | *true* if removed. |
| *get_owner* | *squad*: userdata | string \| nil | Owner name or *nil*. |
| *get_scripted_squads* | — | table | Shallow-copy map keyed by squad id. Fresh per call. |
| *get_scripted_squad* | *squad_id*: number | table \| nil | Shallow copy of one entry. |
| *release_squad* | *squad_id*: number | bool | *false* if AP wasn't scripting it. |
| *script_squad* | *squad*: userdata; *smart*: userdata; *opts*: { *rush*, *on_arrive*, *on_arrive_args*, *pre_release_gulag* } | { code, id, dst, dst_id } | Full lifecycle: engine target + reassert + marker + TTL + gulag. |
| *script_actor_target* | *squad*: userdata | { code, id, dst, dst_id } | Pursue the player; no arrival detection. |
| *add_record* | *subject_squad*: userdata; *cause_key*, *consequence_key*: string; *opts*: { *action_key*, *other_squad*, *smart*, *level_id*, *cause_id* } | number \| nil | *cons_id*. Call after every script_\* SUCCESS or your dispatch is invisible. |
| *get_record* | *opts*: { *squad_id*, *assigned*, ... } | table \| nil | Newest match. O(1) for *{squad_id, assigned=true}*. |
| *get_records* | *opts*: table \| nil | table | Every match (array). |
| *get_text* | *id*: string | string \| nil | Localized phrase from a CONSEQUENCE.X or ACTION.X enum value. |
| *register_radiant_cause* | *category*: CAUSE_CATEGORY.X; *predicate*: function(squad) → table \| nil | bool | Predicate returns publish payload, rejection table, or nil. |
| *register_reactive_cause* | *callback*: CALLBACK.X; *category*: CAUSE_CATEGORY.X; *predicate*: function(...) → table \| nil | bool | Predicate args match the engine callback. |
| *register_consequence* | *name*: string; *opts*: { *cause* (required), *condition*, *on_arrive_fn* }; *handler*: function(event_data) → { code, ... } | bool | Strict; refuses duplicate names. |
| *unregister_consequence* | *name*: string | bool | *true* if removed. |
| *get_cause* | *consequence*: string | string \| nil | Cause string a consequence subscribes to. |

Categories (used by *register_radiant_cause* and *register_reactive_cause*):

| Category | When |
|---|---|
| *CAUSE_CATEGORY.REACTIONS* | World events: NPC died, item used. |
| *CAUSE_CATEGORY.NEEDS* | Squad internal state: eat, sleep. |
| *CAUSE_CATEGORY.INSTINCTS* | Stimulus: scent, sense of danger. |
| *CAUSE_CATEGORY.OPPORTUNITIES* | Environmental state: empty smart, abandoned stash. |

Foreign causes share AP's per-category MCM cap and stats grouping.

---

## Detect AlifePlus

```lua
if not ap_api then return end
```

That's the master gate. Every example below assumes it.

> [!NOTE]
> Per-function presence still matters across versions. *ap_api.add_record* didn't exist before 1.6.0, for example. Check the specific function whenever you call it.

---

## Listen to AP events

Zero AP code dependency — subscribe via xbus. Good for telemetry, journals, sound triggers, anything that reads AP events without participating in the pipeline.

```lua
xbus.subscribe("cause:massacre", function(data)
    -- data.position, data.level_id, data.squad_id, plus payload
    log("massacre at level %d", data.level_id)
end, "my_mod")
```

Cause names are listed in *ap_core_const.CAUSE* — *CAUSE.MASSACRE*, *CAUSE.STASH_LOOT*, *CAUSE.AREA_INFEST*, etc. Each cause's payload shape lives in its generator file (*ap_ext_cause_\**.script and *ap_ext_causes_\**.script).

> [!TIP]
> This path requires only xlibs (xbus). You don't need *ap_api* at all if you only listen.

---

## Coordinate squad ownership

So AP doesn't route to squads your mod controls. Warfare's pattern:

```lua
function on_game_start()
    if not (ap_api and ap_api.register_owner) then return end
    ap_api.register_owner("warfare", function(squad)
        if not squad then return false end
        return squad.registered_with_warfare == true
    end)
end
```

Calling *register_owner* again with the same name replaces the filter. AP excludes owned squads at every layer that routes them: producer gate, cause predicate, find_squads, and the script_squad path.

To stop owning squads (e.g. an MCM toggle that hands them back):

```lua
if ap_api and ap_api.unregister_owner then
    ap_api.unregister_owner("warfare")
end
```

> [!NOTE]
> Ownership respect is gated by MCM *allow_external_ownership*. If a user turns this off, AP ignores all registered owners.

---

## Display AP activity in your UI

A tooltip showing what AP is currently doing with a squad:

```lua
local function ap_action_for(squad_id)
    if not (ap_api and ap_api.get_record and ap_api.get_text) then return nil end
    local rec = ap_api.get_record({ squad_id = squad_id, assigned = true })
    if not rec or not rec.action_key then return nil end
    return ap_api.get_text(rec.action_key)
end

-- Call site:
local text = ap_action_for(squad.id)
if text then tooltip = tooltip .. "\n" .. text end
```

Returns localized strings: "Investigating a Massacre Site", "Heading to a Campfire to Rest", etc. *nil* when AP has no live record for the squad.

The record carries translation keys for both the subject squad and any second party. Build richer tooltips with one *get_text* per key:

```lua
local faction = ap_api.get_text(rec.subject_faction_key)
local species = rec.subject_species_key and ap_api.get_text(rec.subject_species_key)
local name    = rec.subject_name  -- engine-resolved string, no translation needed
-- Same shape for other_faction_key, other_species_key, other_name.
```

> [!TIP]
> *get_record({ squad_id = X, assigned = true })* hits a live index — O(1). Other query shapes are full FIFO scans.

Record entry fields (every translatable field stores a localization key; resolve via *get_text*):

```
cons_id, squad_id, cause_id, other_squad_id, smart_id, level_id
cause_key, consequence_key, action_key
subject_faction_key, subject_species_key, other_faction_key, other_species_key
subject_name, other_name                  -- engine strings, no translation
assigned (bool, true = live), game_hours  -- hour-of-write
```

---

## Take a squad back from AP

When your mod needs a specific squad immediately:

```lua
if ap_api and ap_api.release_squad and ap_api.release_squad(squad.id) then
    -- AP was scripting it; broker's reassert loop is now stopped.
end
xsquad.acquire_squad(squad, your_smart, true)
```

*release_squad* returns *false* if AP wasn't scripting the squad. Safe to call unconditionally.

---

## Drive a squad through AP

When your mod wants AP's full lifecycle (PDA marker, TTL, pre-release gulag, target reassertion) for its own dispatches. *script_squad* sets the engine target; *add_record* makes the dispatch visible to HUD and news.

```lua
if not (ap_api and ap_api.script_squad and ap_api.add_record) then return end

local res = ap_api.script_squad(squad, smart, {
    rush              = true,
    on_arrive         = CONSEQUENCE.MY_MOD_ESCORT,  -- if you registered an arrival handler
    on_arrive_args    = { target_id = target.id },
    pre_release_gulag = 1800,
})

if res.code == ap_core_const.RESULT.SUCCESS then
    ap_api.add_record(squad, CAUSE.MY_MOD_THREAT, CONSEQUENCE.MY_MOD_ESCORT, {
        action_key = ACTION.MY_MOD_ESCORT,
        smart      = smart,
        level_id   = squad.level_id,
    })
end
```

To pursue the player instead of a smart:

```lua
local res = ap_api.script_actor_target(squad)
-- script_actor_target has no opts; it sets scripted_target = "actor" and bypasses arrival detection.
```

> [!IMPORTANT]
> Without an *add_record* call, the dispatch is invisible: no HUD marker, no news, no tooltip. *script_squad* alone only sets engine state.

*script_squad* opts:

```
rush               -- run + danger anim when online
on_arrive          -- dispatch key; must match a consequence registered with on_arrive_fn
on_arrive_args     -- table passed to the arrival handler
pre_release_gulag  -- seconds held after arrival (default from const)
```

*add_record* opts:

```
action_key   -- ACTION.X enum value (player-facing label, localization id)
other_squad  -- optional second-party squad (e.g. revenge target)
smart        -- destination smart server object (id stored; resolved lazily at render)
level_id     -- level the event happened on
cause_id     -- optional correlator if your cause publishes sibling consequences
```

---

## Add a cause and consequence

Full pipeline participation. Your cause goes through AP's gates (per-category rate limit, alife_ratio, distributor), publishes via xbus, and is consumed by your registered consequence handler. The framework handles rate limiting, tracing, arrival, marker lifecycle, and cleanup.

```lua
local CAUSE_AMBUSH             = "cause:my_mod:ambush"
local CONSEQUENCE_AMBUSH_SETUP = "consequence:my_mod_ambush_setup"
local ACTION_AMBUSH_SETUP      = "action:my_mod_ambush_setup"
local CATEGORY = ap_core_const.CAUSE_CATEGORY.OPPORTUNITIES
local cfg = my_mod_mcm.cfg

-- Cause predicate: outlaws notice an empty smart, want to ambush
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

-- Arrival handler: log when a dispatched squad reaches the smart
local function _on_arrive(squad, args)
    log("squad %d arrived to ambush at smart %d", squad.id, args.smart_id)
end

-- Consequence handler: route nearby outlaws to the smart
local function _handler(event_data)
    local smart = xobject.se(event_data.smart_id)
    if not smart then return { code = ap_core_const.RESULT.FAILED_SCAN } end
    local squads = find_outlaw_squads_near(event_data)
    for _, squad in ipairs(squads) do
        local res = ap_api.script_squad(squad, smart, {
            on_arrive         = CONSEQUENCE_AMBUSH_SETUP,
            on_arrive_args    = { smart_id = event_data.smart_id },
            rush              = true,
            pre_release_gulag = 1800,
        })
        if res.code == ap_core_const.RESULT.SUCCESS and ap_api.add_record then
            ap_api.add_record(squad, CAUSE_AMBUSH, CONSEQUENCE_AMBUSH_SETUP, {
                action_key = ACTION_AMBUSH_SETUP,
                smart      = smart,
                level_id   = event_data.level_id,
            })
        end
    end
    return { code = ap_core_const.RESULT.SUCCESS }
end

function on_game_start()
    if not (ap_api and ap_api.register_radiant_cause and ap_api.register_consequence) then return end
    ap_api.register_radiant_cause(CATEGORY, _predicate)
    ap_api.register_consequence(CONSEQUENCE_AMBUSH_SETUP, {
        cause        = CAUSE_AMBUSH,
        condition    = function() return cfg.ambush_setup_enabled end,
        on_arrive_fn = _on_arrive,
    }, _handler)
end
```

Conventions:

- Cause strings: namespace as *"cause:&lt;modname&gt;:&lt;event&gt;"* to avoid collision with AP's standard causes.
- Consequence names: namespace as *"consequence:&lt;modname&gt;_&lt;event&gt;"* so the consequence enum value is itself a valid localization id.
- Pass the consequence registration name as *on_arrive* if you want the consequence to handle its own arrivals.
- Always call *add_record* after a SUCCESS dispatch. Without a record, no marker is shown and news cannot narrate the event.
- Self-gate against the per-category cause budget with *ap_core_limiter.check_cause_rate_limit* and *increment_cause_counter*. The framework does not gate this for you.
- Predicate payload must include *squad_id* and *level_id* (AP debug log reads them).

Predicate return shapes:

```lua
-- Publish (cause fires through the pipeline):
return {
    cause     = "cause:<modname>:<event>",  -- routing key, must be unique
    squad_id  = squad.id,                   -- required: AP debug log reads it
    level_id  = squad.level_id,             -- required: AP debug log reads it
    -- ...any other fields your consequence needs
}

-- Structured rejection (for telemetry):
return { code = RESULT.FAILED_RULES, reason = REASON.WRONG_ALIGNMENT }

-- Silent reject:
return nil
```

Reactive predicates have the same return shape but the argument list matches the engine callback's args (use *register_reactive_cause(callback, category, predicate)*).

---

## Localization

*CONSEQUENCE.X* and *ACTION.X* enum values are themselves localization ids:

```
CONSEQUENCE.MASSACRE_INVESTIGATE = "consequence:massacre_investigate"
ACTION.MASSACRE_INVESTIGATE      = "action:massacre_investigate"
```

Strings live in *gamedata/configs/text/eng/ui_st_ap_consequence.xml* and *gamedata/configs/text/rus/ui_st_ap_consequence.xml*. *get_text* is *game.translate_string* under the hood — it works for any localization key on the record (action, consequence, faction, species).

Foreign mods adding new consequences add matching *&lt;string id="..."&gt;* entries in their own *ui_st_&lt;modname&gt;.xml* (eng + rus).

> [!WARNING]
> The Russian XML is windows-1251. Edit it via *xmlstarlet* only. Edit/Write/sed corrupt Cyrillic to U+FFFD.

---

## Notes

- *ap_api* is the contract. *ap_core_\** modules are internal.
- Listener-only mods (no pipeline participation) can use raw xbus.subscribe without touching *ap_api* at all.

---

## Contact

- Discord: **damian_sirbu**
- ModDB: **damian_sirbu**
- Email: **dami.sirbu@gmail.com**
- GitHub: **github.com/damiansirbu-stalker**
