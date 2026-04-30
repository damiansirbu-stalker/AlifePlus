# AlifePlus Integration Guide

Three integration levels, from passive listener to full pipeline participant.

Requires: xlibs 1.4.0+, AlifePlus loaded. See `architecture.md` for internals, `conventions.md` for naming.

---

## Integration Levels

| Level | What you do | What you get | Dependency |
|-------|------------|--------------|------------|
| 1. Listen | Subscribe to xbus events | Cause data (position, level, entities) | xlibs only |
| 2. Register | Register cause predicate + consequence handler | Protection, rate limiting, PDA, markers, lifecycle | ap_core_* |
| 3. Coordinate | Use tracker, conquest, or broker internals | Domain state, territory control, squad ownership | ap_core_* + ap_ext_* |

---

## Register: Add a Cause + Consequence

The fastest way to extend AP. Register a predicate and a handler. The framework handles gates, protection, rate limiting, tracing, PDA routing, squad lifecycle.

---

### Cause (predicate)

A cause evaluates a world-state condition and returns structured data on match, or a rejection table otherwise.

```lua
-- ap_ext_cause_ambush.script
local CAUSE = ap_core_const.CAUSE
local CAUSE_TYPE = ap_core_const.CAUSE_TYPE
local RESULT = ap_core_const.RESULT
local REASON = ap_core_const.REASON
local cfg = ap_core_mcm.cfg

local function _predicate(trace, squad)
    if not cfg.cause_ambush_enabled then return { code = RESULT.FAILED_RULES } end

    local level_id = xlevel.get_level_id(squad)
    if not level_id then return { code = RESULT.FAILED_RULES, reason = REASON.NO_LEVEL_ID } end

    local smart = ap_core_utils.find_smart(squad.position, {
        level_id = level_id, max_distance = ap_core_const.RANGE_EYE,
        filter = xsmart.is_smart_empty,
    })
    if not smart then return { code = RESULT.FAILED_RULES, reason = REASON.NO_SMART } end

    return {
        cause = CAUSE.AMBUSH,
        squad_id = squad.id,
        community = squad.player_id,
        smart_id = smart.id,
        level_id = level_id,
        position = squad.position,
    }
end

function on_game_start()
    local cbs = ap_core_const.RADIANT_CALLBACKS
    for i = 1, #cbs do
        ap_core_producer.register({
            callback = cbs[i],
            cause_type = CAUSE_TYPE.RADIANT,
            category = CAUSE_CATEGORY.OPPORTUNITIES,
        }, _predicate)
    end
end
```

Rules:
- Return `{ cause = CAUSE.X, ...payload }` on match, `{ code = RESULT.FAILED_RULES, ... }` on reject
- Each call publishes one specific cause; no umbrella names
- Predicates self-observe (prof + trace:push + debug log) — never call `observe()` or manipulate counters
- Include `level_id` from the source entity, never the actor
- Include `community = squad.player_id` for downstream alignment/personality checks
- Add `CAUSE.AMBUSH = "cause:ambush"` to ap_core_const
- Add MCM `cause_ambush_enabled` to ap_core_mcm.defaults
- `config.category` is required. Pick from `CAUSE_CATEGORY.REACTIONS / NEEDS / INSTINCTS / OPPORTUNITIES` based on what your cause represents.

---

### Consequence (handler)

A consequence follows the three-phase template: rules -> eval -> action. Each phase returns immediately on failure with the corresponding result code.

```lua
-- ap_ext_consequence_ambush_setup.script
local RESULT = ap_core_const.RESULT
local REASON = ap_core_const.REASON
local CONSEQUENCE = ap_core_const.CONSEQUENCE
local CAUSE = ap_core_const.CAUSE
local PHASE = ap_core_const.CONSEQUENCE_PHASE
local PRE_RELEASE_GULAG = ap_core_const.PRE_RELEASE_GULAG
local PERSONALITY = ap_ext_const.PERSONALITY
local cfg = ap_core_mcm.cfg

local _alignment = ap_ext_const.alignment_outlaw
local _personality = { PERSONALITY.AGGRESSION, PERSONALITY.GREED }

local function _on_arrive(squad, args)
    -- runs when squad reaches destination smart
    -- args contains on_arrive_args from script_squad opts
end

local function _handler(event_data)
    local trace = event_data._trace
    return ap_core_debug.observe(trace, CONSEQUENCE.AMBUSH_SETUP, function()

        -- RULES: alignment, personality, validation
        if not ap_ext_util.check_alignment(_alignment, event_data.community) then
            return { code = RESULT.FAILED_RULES, reason = REASON.WRONG_ALIGNMENT }
        end
        if not ap_ext_util.check_personality(_personality, event_data.community,
            CONSEQUENCE.AMBUSH_SETUP, cfg.consequence_ambush_setup_personality_min,
            cfg.consequence_ambush_setup_personality_max) then
            return { code = RESULT.FAILED_RULES, reason = REASON.LOW_PERSONALITY }
        end

        -- EVAL: world queries
        local smart = xobject.se(event_data.smart_id)
        if not smart then return { code = RESULT.FAILED_EVAL, reason = REASON.NO_SMART } end

        local squads = ap_core_utils.find_squads_observed(trace, event_data.position, {
            factions = _alignment,
            level_id = event_data.level_id,
            max_distance = ap_core_const.RANGE_RADIO,
            max_count = cfg.consequence_ambush_setup_max_squads,
            exclude_at_smart_id = smart.id,
        })
        if #squads == 0 then return { code = RESULT.FAILED_EVAL, reason = REASON.NO_SQUAD } end

        -- ACTION: script squads, record activity
        local moved = {}
        ap_core_debug.observe(trace, PHASE.MOVE_SQUAD, function()
            for i = 1, #squads do
                local res = ap_core_broker.script_squad(squads[i], smart, {
                    rush = cfg.consequence_ambush_setup_rush,
                    on_arrive = CONSEQUENCE.AMBUSH_SETUP,
                    pre_release_gulag = PRE_RELEASE_GULAG.AMBUSH_SETUP,
                })
                if res.code == RESULT.SUCCESS then moved[#moved + 1] = squads[i] end
            end
            return ap_core_debug.result_squads(moved, { dst_id = smart.id })
        end)
        if #moved == 0 then return { code = RESULT.FAILED_ACTION, reason = REASON.MOVE_FAILED } end

        for i = 1, #moved do
            ap_ext_news.record_event(moved[i], CAUSE.AMBUSH, CONSEQUENCE.AMBUSH_SETUP, {
                cause_id = event_data._cause_id,
                smart = smart, level_id = event_data.level_id,
            })
        end

        return ap_core_debug.result_squads(moved, { code = RESULT.SUCCESS, dst_id = smart.id })
    end)
end

function on_game_start()
    ap_core_consumer.register(CONSEQUENCE.AMBUSH_SETUP, {
        event = CAUSE.AMBUSH,
        condition = function() return cfg.consequence_ambush_setup_enabled end,
        on_arrive = _on_arrive,
    }, _handler)
end
```

Rules:
- Always return `{ code = RESULT.X }` where X is SUCCESS, FAILED_RULES, FAILED_EVAL, or FAILED_ACTION
- Follow the consequence template: rules -> eval -> action (see conventions.md)
- Enabled gate goes in `condition` function (consumer pre-gate), not inside the handler
- Personality checked inside handler via `ap_ext_util.check_personality`
- Wrap the handler body in `ap_core_debug.observe(trace, CONSEQUENCE.X, function() ... end)`
- Wrap actions (move) in their own `observe` calls for structured tracing
- Call `ap_ext_news.record_event()` after successful script_squad (composer dispatches PDA gossip from the journal; no per-consequence PDA call needed)
- Use `find_squads_observed` for traced search (protections applied automatically)
- Add `CONSEQUENCE.AMBUSH_SETUP = "consequence:ambush_setup"` to ap_core_const

---

### Chase (re-script on arrival)

For consequences that pursue a moving target, the arrival handler re-scripts the squad to the target's new location. Each re-script increments a chase counter. The squad gives up after `max_chases`.

```lua
local function _on_arrive(squad, args)
    if not args or not args.target_squad_id then return end
    local chase_count = (args.chase_count or 0) + 1
    if chase_count > cfg.consequence_ambush_setup_max_chases then return end

    local target_squad = xobject.se(args.target_squad_id)
    if not target_squad then return end

    local level_id = xlevel.get_level_id(target_squad)
    if not level_id or xlevel.get_level_id(squad) ~= level_id then return end

    local smart = ap_core_utils.find_smart(target_squad.position, {
        level_id = level_id, max_distance = ap_core_const.RANGE_RADIO,
    })
    if not smart then return end

    ap_core_broker.script_squad(squad, smart, {
        rush = cfg.consequence_ambush_setup_rush,
        on_arrive = CONSEQUENCE.AMBUSH_SETUP,
        on_arrive_args = { target_squad_id = args.target_squad_id, chase_count = chase_count },
        pre_release_gulag = PRE_RELEASE_GULAG.AMBUSH_SETUP,
    })
end
```

Link arrival to squad: pass `on_arrive = CONSEQUENCE.AMBUSH_SETUP` in `script_squad` opts. The broker matches the key to the handler registered via `consumer.register(..., { on_arrive = _on_arrive })`.

---

### Checklist

1. Add `CAUSE.X` to ap_core_const CAUSE table
2. Add `CONSEQUENCE.X_VERB` to ap_core_const CONSEQUENCE table
3. Add MCM defaults to ap_core_mcm.defaults
4. Create `ap_ext_cause_x.script` with predicate, register in on_game_start
5. Create `ap_ext_consequence_x_verb.script` with handler, register in on_game_start
6. (Optional) Add 5 templates at `st_ap_news_tpl_<consequence>_001..005` in `gamedata/configs/text/eng/ui_st_ap_news.xml` so the consequence surfaces in PDA news
7. Add MCM UI entries to ap_core_mcm menu builder
8. Run validator: `stalker-manager.sh validate`

---

## Two Alife Mods Collaborating

ModA controls squads for territorial warfare. AP controls squads for emergent behavior. Neither knows the other's internals.

---

### Squad coordination

`scripted_target` is the squad control field. Setting it routes the squad to `specific_update` (direct A->B movement instead of simulation). `xsquad` provides three primitives:

```lua
xsquad.control_squad(squad, smart, rush)   -- acquire: sets scripted_target, clears __lock
xsquad.release_squad(squad)                -- release: clears scripted_target + __lock
xsquad.reassert_target(squad, target)      -- defend: restores scripted_target if overwritten
```

Both mods check `scripted_target` before claiming a squad. This is enough for basic non-interference:

```lua
-- ModA before scripting:
if se_squad.scripted_target then return end  -- AP (or anyone) has this squad
xsquad.control_squad(squad, smart)

-- AP's PROTECTION gate already does this check automatically.
-- No code change needed on AP's side.
```

AP reasserts `scripted_target` on its squads every 20s. Squads that die or despawn between scans are removed from AP's tracking table automatically.

---

### Ownership registry (identity)

For knowing WHO controls a squad (not just that it's controlled), register an ownership filter:

```lua
-- In ModA's on_game_start:
ap_core_broker.register_owner("warfare", function(squad)
    return squad.registered_with_warfare == true
end)
```

AP already registers warfare and BAO in `ap_core_compat`:

```lua
ap_core_broker.register_owner("warfare", function(squad)
    return squad.registered_with_warfare == true
end)
ap_core_broker.register_owner("bao", function(squad)
    return squad.__lock == true
end)
```

On name match, `register_owner` replaces the existing filter. This lets warfare register natively and override AP's proxy from `ap_core_compat`.

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

## Listen: Subscribe to Events

Subscribe to cause events via xbus. No AP dependency -- xlibs only.

```lua
function on_game_start()
    xbus.subscribe("cause:massacre", function(e)
        printf("massacre at level %s: %d dead, faction %s",
            e.level_id, e.total_deaths, e.faction)
    end, "my_mod_listener")
end
```

---

### Events

| Event | Type | Key payload |
|-------|------|-------------|
| cause:massacre | reactive | position, level_id, smart, total_deaths, faction |
| cause:squadkill | reactive | position, level_id, squad_id, killer_id, victim_faction |
| cause:basekill | reactive | position, level_id, smart, factions, faction, death_count |
| cause:alpha | reactive | killer_id, species, kills, level, level_id |
| cause:alphakill | reactive | id, killer_id, community, name, alpha_level, kills |
| cause:wounded | reactive | position, level_id, npc_id, squad_id, is_player, health |
| cause:harvest | reactive | position, level_id, taker_id, taker_squad_id, artefact_section |
| cause:stash_loot | radiant | position, level_id, squad_id, community, stash_id, stash_position, items_count, dto_field |
| cause:stash_ambush | radiant | position, level_id, squad_id, community, stash_id, stash_position, items_count, dto_field |
| cause:stash_fill | radiant | position, level_id, squad_id, community, stash_id, stash_position, items_count, dto_field |
| cause:area_conquer | radiant | position, level_id, squad_id, community, dto_field |
| cause:area_infest | radiant | position, level_id, squad_id, community, species, dto_field |
| cause:hunger / cause:sleep / cause:rest / cause:heal / cause:shelter / cause:supply / cause:money / cause:job / cause:social | radiant | position, level_id, squad_id, community, dto_field, drive |
| cause:instinct_scatter / cause:instinct_feed / cause:instinct_sleep / cause:instinct_explore / cause:instinct_socialize | radiant | position, level_id, squad_id, community, species, dto_field, drive |

All events include `_trace`, `_cause_id` (broker-assigned monotonic seq), and `cause_type` ("radiant" or "reactive"). The umbrella names (`cause:stash`, `cause:area`, `cause:needs`, `cause:instincts`) are **never published** — only the specific child causes above flow through xbus.

---

### Use cases

- Faction relations mod: adjust goodwill on massacre/basekill
- Statistics tracker: count causes per level, per faction
- Sound mod: play ambient audio when nearby events fire
- Journal mod: log events to a player-readable journal

---

## Coordinate: Deep Integration

Direct access to AP domain systems. APIs may change between versions.

---

### Tracker (ap_ext_tracker)

| Function | Returns |
|----------|---------|
| `get_alpha(entity_id)` | alpha data table { level, kills, name, level_id } or nil |
| `get_alpha_level(entity_id)` | integer level or 0 |
| `is_alpha(entity_id)` | boolean (true even during death grace period) |
| `get_alphas()` | all alive alphas as { [npc_id] = data } |
| `get_stalker_needs(squad_id)` | needs DTO { last_hunger_at, last_sleep_at, ... } or nil |
| `get_mutant_instincts(squad_id)` | instinct DTO { last_feed_at, last_sleep_at, ... } or nil |
| `projected_kill_count(killer_id, victim_id)` | kill count inclusive of a pending death |

---

### Smart Mutator (ap_ext_smart_mutator)

| Function | Returns |
|----------|---------|
| `conquer_smart(smart_id, faction)` | set faction ownership with FIFO eviction and decay |

---

### Broker (ap_core_broker)

| Function | Returns |
|----------|---------|
| `script_squad(squad, smart, opts)` | { code, id, dst, dst_id } -- script with lifecycle |
| `script_actor_target(squad)` | { code, id, dst, dst_id } -- pursue player (no arrival) |
| `is_protected(squad)` | boolean, reason, detail, name -- check all guards |
| `get_owner(squad)` | string or nil -- ownership query |
| `register_owner(name, filter_fn)` | register ownership filter (replaces on name match) |
| `record(squad_id, cause, consequence, opts)` | append to activity FIFO (markers, composer, external queries) |
| `get_record(opts)` | return the most-recent record matching `opts` (AND-logic) or nil. `{ squad_id, assigned = true }` is O(1) |
| `get_records(opts)` | return array of records matching `opts` (AND-logic). `{ assigned = true }` reads live entries only |
| `clear_record(squad_id)` | clear assigned entry on entity death (pure entry drop -- broker stays out of marker lifecycle) |
| `register_arrival_handler(key, fn)` | register on-arrival callback by key; consumer.register also accepts an `on_arrive` opt that wires this for you |
| `get_scripted_ids()` | read-only reference to _ap_scripted_squads table |

---

### Const (ap_core_const)

Read-only static tables and enums that integrators reference directly. No registration needed.

| Symbol | Use |
|--------|-----|
| `CAUSE` | enum of cause event names (e.g. `CAUSE.MASSACRE` -> `"cause:massacre"`). Use as xbus event keys. |
| `CONSEQUENCE` | enum of consequence keys (e.g. `CONSEQUENCE.MASSACRE_INVESTIGATE`). Match to `record.consequence`. |
| `CONSEQUENCE_INFO` | per-consequence `{ name_key, action_key }`. `name_key` is the short caption ("Massacre Investigate"); `action_key` is the full action phrase ("Investigating a Massacre Site"). Both XML ids resolved via `game.translate_string`. |
| `CONSEQUENCE_PHASE` | trace-only enum used by `observe()` for sub-phase paths (FIND_TARGETS, MOVE_SQUAD, ARRIVE, etc.). Integrators rarely need this; it shows up in DEBUG-level traces. |
| `RESULT` | consequence handler return codes (SUCCESS, FAILED_RULES, FAILED_EVAL, FAILED_ACTION, DISABLED). |
| `REASON` | reason codes for skip/failure traces (NIL_ARGS, IS_OWNED, NO_SQUAD, ...). |
| `RANGE_EYE` / `RANGE_RADIO` / `RANGE_SCENT` | range tier constants in meters. Use when calling `ap_core_utils.find_squads`/`find_smart`. |

`script_squad` does not check protection. Caller must verify:

```lua
local protected = ap_core_broker.is_protected(squad)
if protected then return end
ap_core_broker.script_squad(squad, smart, {
    rush = true,
    on_arrive = CONSEQUENCE.MY_CONSEQUENCE,
    on_arrive_args = { target_id = target.id },
    pre_release_gulag = 1800,
})
```

---

### Activity Record

After a consequence scripts a squad, AP appends an entry to a bounded FIFO (capacity 256). The broker stores ids, event-time facts, and squad-derived display strings (faction key + localized display, commander name, species + localized plural — eagerly captured at record time so dead/unregistered squads stay renderable). Smart and level remain lazy. External mods query the squad's currently-assigned record via `get_record({ squad_id, assigned = true })` and render the localized action phrase via `ap_core_const.CONSEQUENCE_INFO[record.consequence].action_key` + `game.translate_string()`.

Schema (per entry):

| Field | Type | Notes |
|-------|------|-------|
| `squad_id` | number | subject squad server id |
| `other_squad_id` | number or nil | opposing party (killer, bleeder, taker) in two-party events |
| `cause` | string | CAUSE.X enum (e.g. `"cause:massacre"`) |
| `consequence` | string | CONSEQUENCE.X enum (e.g. `"consequence:massacre_investigate"`) |
| `cons_id` | number | monotonic sequence id assigned by broker |
| `cause_id` | number or nil | shared by siblings of one cause publish; groups siblings for news dispatch |
| `smart_id` | number or nil | acting smart terrain (resolve display lazily via `xlevel.get_smart_display_name(xobject.se(smart_id))`) |
| `level_id` | number or nil | acting level (resolve display lazily via `xlevel.get_level_name(level_id)`) |
| `game_hours` | number | event-time snapshot (hours) |
| `subject_faction` | string or nil | engine faction key (e.g. `"stalker"`, `"monster_predatory_day"`) |
| `subject_faction_display` | string or nil | localized faction (`"Loners"`, `"predators"`); use directly in UI text |
| `subject_name` | string or nil | commander's character name; nil for empty / mutant / unbound squads |
| `subject_species` | string or nil | xcreature species (`"bloodsucker"`); nil for stalker squads |
| `subject_species_display` | string or nil | localized species plural (`"bloodsuckers"`); nil if species not in `SPECIES_DISPLAY_KEY` |
| `other_faction`, `other_faction_display`, `other_name`, `other_species`, `other_species_display` | string or nil | same five fields for `other_squad`; all nil for single-actor consequences |
| `assigned` | boolean | true while this is the squad's current entry; flips to false on supersede or clear |

Display strings are captured at the moment the entry is written. Locale switches mid-session leave older entries in the previous locale until natural FIFO churn (~10-50 minutes); switches are rare and self-correcting.

When a new entry is recorded for the same squad, the previous entry's `assigned` flag is set to false. Map markers only display `assigned` entries. The FIFO is session-lifetime (resets on load, no save persistence).

---

### Warfare map tooltip: display AP activity

Goal: when your warfare map UI hovers a squad, append what the squad is doing according to AlifePlus ("Investigating a Massacre Site", "Guarding an Outpost", "Hunting the Wounded"). Strings are localized. Both English and Russian ship with AP. Copy-paste example, warfare author edits only the call site at the bottom:

```lua
--- Return localized AP action phrase for a squad, or nil if nothing to show.
--- Graceful no-op when AlifePlus is absent, squad is owned by warfare, or squad has no record.
local function get_ap_action(squad_id)
    if not ap_core_broker or not ap_core_const then return nil end

    local record = ap_core_broker.get_record({ squad_id = squad_id, assigned = true })
    if not record then return nil end

    local info = ap_core_const.CONSEQUENCE_INFO[record.consequence]
    if not info then return nil end

    local text = game.translate_string(info.action_key)
    if not text or text == info.action_key or text == "" then return nil end
    return text
end

-- Call site (warfare's tooltip builder, wherever you assemble the hover text):
local ap_action = get_ap_action(squad.id)
if ap_action then
    tooltip_text = tooltip_text .. "\n" .. ap_action
end
```

Notes for warfare authors:

- Call `get_record` fresh on every render tick. Records mutate as squads move between consequences.
- `ap_core_const.CONSEQUENCE_INFO` is a static const table — read it directly, don't cache.
- Each entry has both `action_key` (full phrase shown to the player) and `name_key` (short caption for config / debug). Pick whichever fits your UI.
- Locale switch (English/Russian) works automatically because the resolve happens at render time via `game.translate_string`.
- A nil return means the squad has no recent AP activity (or is warfare-owned). Render nothing.
- Squads warfare owns are already excluded from AP via the ownership registry (`ap_core_broker.register_owner("warfare", ...)`, registered by default). They get no record, so `get_record` returns nil, so you render nothing. No additional filtering needed on warfare's side.

Typical action outputs (EN): "Investigating a Massacre Site", "Scavenging a Massacre Site", "Reinforcing Attacked Base", "Evacuating Attacked Base", "Hunting the Wounded", "Guarding an Outpost", "Out Exploring", "Heading to a Campfire to Rest", "Restocking at Trader", "Harvesting Artefacts". 36 entries total, one per AP consequence. Full map: `ap_core_const.CONSEQUENCE_INFO` keys.

---

### Warfare news: suppress AP messages for warfare-owned squads

AP's PDA news surfaces activity for squads AP controls. Warfare-owned squads are already skipped at the protection gate (via the default `register_owner("warfare", ...)` registration), so they never enter the news pipeline. If warfare overrides the default filter (to scope warfare ownership more precisely), the same registration handles the suppression:

```lua
-- In warfare's on_game_start, after warfare's own ownership markers are defined:
ap_core_broker.register_owner("warfare", function(squad)
    return squad.registered_with_warfare == true  -- warfare's existing flag
end)
```

`register_owner` replaces the existing filter on name match. This lets warfare scope ownership natively and override AP's default proxy from `ap_core_compat`. After registration, AP excludes matching squads at four layers (producer gate, cause predicate, `find_squads`, and squad scripting), so warfare-owned squads are fully invisible to AP.

---

## Contact

Questions, integration requests, API feedback:

- Discord: **damian_sirbu**
- ModDB: **damian_sirbu**
- Email: **dami.sirbu@gmail.com**
- GitHub: **github.com/damiansirbu-stalker**
