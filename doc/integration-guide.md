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
local CAUSE_CATEGORY = ap_ext_const.CAUSE_CATEGORY
local CATEGORY = CAUSE_CATEGORY.OPPORTUNITIES
local CAP_KEY = "cause_max_opportunities"
local cfg = ap_core_mcm.cfg

local function _predicate(trace, squad)
    if not cfg.cause_ambush_enabled then return { code = RESULT.FAILED_RULES } end
    if not ap_core_limiter.check_cause_rate_limit(CATEGORY, cfg[CAP_KEY]) then
        return { code = RESULT.FAILED_RULES, reason = REASON.BUDGET_EXHAUSTED }
    end

    local level_id = xlevel.get_level_id(squad)
    if not level_id then return { code = RESULT.FAILED_RULES, reason = REASON.NO_LEVEL_ID } end

    local smart = ap_core_util.find_smart(squad.position, {
        level_id = level_id, max_distance = ap_core_const.RANGE_EYE,
        filter = xsmart.is_smart_empty,
    })
    if not smart then return { code = RESULT.FAILED_RULES, reason = REASON.NO_SMART } end

    ap_core_limiter.increment_cause_counter(CATEGORY)
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
            category = CATEGORY,
        }, _predicate)
    end
end
```

Rules:
- Return `{ cause = CAUSE.X, ...payload }` on match, `{ code = RESULT.FAILED_RULES, ... }` on reject
- Each call publishes one specific cause; no umbrella names
- Predicates self-observe (prof + trace:push + debug log) -- never call `observe()` or manipulate framework state beyond the cause counter
- Self-gate against the per-category cause budget via `ap_core_limiter.check_cause_rate_limit(CATEGORY, cfg[CAP_KEY])` and increment via `increment_cause_counter(CATEGORY)` before returning the published payload. The producer does not gate cause budgets.
- Include `level_id` from the source entity, never the actor
- Include `community = squad.player_id` for downstream alignment / personality checks
- Add `CAUSE.AMBUSH = "cause:ambush"` to ap_core_const
- Add MCM `cause_ambush_enabled` and `cause_max_opportunities` (or whichever category cap key applies) to ap_core_mcm.defaults
- `config.category` is required. Pick from `CAUSE_CATEGORY.REACTIONS / NEEDS / INSTINCTS / OPPORTUNITIES`. `CAUSE_CATEGORY` lives in `ap_ext_const`.

---

### Consequence (handler)

A consequence follows the three-phase template: rules -> scan -> action. Each phase returns immediately on failure with the corresponding result code.

```lua
-- ap_ext_consequences_ambush.script
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
            { name = CONSEQUENCE.AMBUSH_SETUP }) then
            return { code = RESULT.FAILED_RULES, reason = REASON.LOW_PERSONALITY }
        end

        -- SCAN: world queries
        local smart = xobject.se(event_data.smart_id)
        if not smart then return { code = RESULT.FAILED_SCAN, reason = REASON.NO_SMART } end

        local squads = ap_core_util.find_squads_observed(trace, event_data.position, {
            factions = _alignment,
            level_id = event_data.level_id,
            max_distance = ap_core_const.RANGE_RADIO,
            max_count = cfg.consequence_ambush_setup_max_squads,
            exclude_at_smart_id = smart.id,
        })
        if #squads == 0 then return { code = RESULT.FAILED_SCAN, reason = REASON.NO_SQUAD } end

        -- ACTION: script squads, record activity
        local moved = {}
        ap_core_debug.observe(trace, PHASE.MOVE_SQUAD, function()
            for i = 1, #squads do
                local res = ap_core_broker.script_squad(squads[i], smart, {
                    consequence = CONSEQUENCE.AMBUSH_SETUP,
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
            ap_ext_news.add(moved[i], CAUSE.AMBUSH, CONSEQUENCE.AMBUSH_SETUP, {
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
- Always return `{ code = RESULT.X }` where X is SUCCESS, FAILED_RULES, FAILED_SCAN, or FAILED_ACTION
- Follow the consequence template: rules -> scan -> action (see conventions.md)
- Enabled gate goes in `condition` function (consumer pre-gate), not inside the handler
- Personality checked inside handler via `ap_ext_util.check_personality`
- Wrap the handler body in `ap_core_debug.observe(trace, CONSEQUENCE.X, function() ... end)`
- Wrap actions (move) in their own `observe` calls for structured tracing
- Call `ap_ext_news.add()` after successful `script_squad`. News owns its own session FIFO; the composer drains it on a randomized interval and dispatches PDA gossip via `dynamic_news_manager`. No per-consequence PDA call. `add()` rolls `cfg.news_chance` (1-100, default 20) and drops silently on miss; pass `opts.other_squad` for two-party events so templates can render the other side.
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

    local smart = ap_core_util.find_smart(target_squad.position, {
        level_id = level_id, max_distance = ap_core_const.RANGE_RADIO,
    })
    if not smart then return end

    ap_core_broker.script_squad(squad, smart, {
        consequence = CONSEQUENCE.AMBUSH_SETUP,
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
4. Create `ap_ext_cause_<family>.script` (singular: generator publishes one cause) or `ap_ext_causes_<family>.script` (plural: generator publishes multiple) with predicate, register in `on_game_start`
5. Create `ap_ext_consequences_<family>.script` (always plural) with handler(s), register in on_game_start
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
xsquad.acquire_squad(squad, smart, rush)   -- acquire: sets scripted_target, clears __lock
xsquad.release_squad(squad)                -- release: clears scripted_target + __lock
xsquad.reassert_target(squad, target)      -- defend: restores scripted_target if overwritten
```

Both mods check `scripted_target` before claiming a squad. This is enough for basic non-interference:

```lua
-- ModA before scripting:
if se_squad.scripted_target then return end  -- AP (or anyone) has this squad
xsquad.acquire_squad(squad, smart)

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

AP already registers warfare in `ap_core_compat`:

```lua
ap_core_broker.register_owner("warfare", function(squad)
    return squad.registered_with_warfare == true
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
            e.level_id, e.kill_count, e.faction)
    end, "my_mod_listener")
end
```

---

### Events

| Event | Type | Key payload |
|-------|------|-------------|
| cause:massacre | reactive | position, level_id, smart, kill_count, faction |
| cause:squadkill | reactive | position, level_id, squad_id, killer_id, victim_faction |
| cause:basekill | reactive | position, level_id, smart, factions, faction, kill_count |
| cause:alpha | reactive | killer_id, species, kills, level, level_id |
| cause:alphakill | reactive | id, killer_id, community, species, killer_is_player, position, level_id |
| cause:wounded | reactive | position, level_id, npc_id, squad_id, is_player, health |
| cause:harvest | reactive | position, level_id, taker_id, taker_squad_id, artefact_section |
| cause:stash_loot | radiant | position, level_id, squad_id, community, stash_id, stash_position, items_count, dto_field |
| cause:stash_ambush | radiant | position, level_id, squad_id, community, stash_id, stash_position, items_count, dto_field |
| cause:stash_fill | radiant | position, level_id, squad_id, community, stash_id, stash_position, items_count, dto_field |
| cause:area_conquer | radiant | position, level_id, squad_id, community, dto_field |
| cause:area_swarm | radiant | position, level_id, squad_id, community, species, dto_field |
| cause:area_infest | radiant | position, level_id, squad_id, community, species, dto_field |
| cause:hunger_campfire / cause:sleep_campfire / cause:rest_campfire / cause:heal_shelter / cause:shelter_indoor / cause:shelter_outdoor / cause:supply_trader / cause:money_harvest / cause:money_hunt / cause:job_outpost / cause:job_explore / cause:job_research / cause:social_campfire / cause:social_base | radiant | position, level_id, squad_id, community, dto_field, drive |
| cause:scatter / cause:feed / cause:slumber_field / cause:slumber_lair / cause:slumber_surge / cause:roam / cause:pack | radiant | position, level_id, squad_id, community, species, dto_field, drive |

All events include `_trace` and `cause_type` (`"radiant"` or `"reactive"`). The umbrella names (`cause:stash`, `cause:area`, `cause:needs`, `cause:instincts`) are **never published**: only the specific child causes above flow through xbus.

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
| `get_alpha(entity_id)` | alpha data table `{ level, kills }` or nil |
| `get_alpha_level(entity_id)` | integer level or 0 |
| `is_alpha(entity_id)` | boolean (true even during death grace period) |
| `get_alphas()` | all alive alphas as `{ [npc_id] = data }` |
| `get_alpha_marker_squads()` | array of unique squad ids hosting alive alphas (registered as the HUD identity source) |
| `get_stalker_needs(squad_id)` | per-squad needs DTO (9 `last_<need>_at` timestamps) or nil |
| `get_mutant_instincts(squad_id)` | per-squad instinct DTO (5 `last_<instinct>_at` timestamps) or nil |
| `get_squad_opportunities(squad_id)` | per-squad opportunity DTO (6 MVT `last_<opportunity>_at` timestamps for stash and area causes) or nil |
| `projected_kill_count(killer_id, victim_id)` | kill count inclusive of a pending death |

---

### Smart Mutator (ap_ext_smart_mutator)

| Function | Returns |
|----------|---------|
| `conquer_smart(smart_id, faction)` | shared spawn injection (LTX entries continue alongside). FIFO eviction at `cfg.area_conquest_max_smarts`, decay after `cfg.area_conquest_decay_hours` |
| `infest_smart(smart_id, faction, level_id)` | exclusive spawn injection (LTX entries suppressed via faction gate). Per-level cap `cfg.area_infest_max_per_level` |
| `is_infested(smart_id)` | boolean |
| `can_infest_on_level(level_id)` | boolean (per-level cap not exceeded) |

---

### Broker (ap_core_broker)

| Function | Returns |
|----------|---------|
| `script_squad(squad, smart, opts)` | `{ code, id, dst, dst_id }` - script with full lifecycle |
| `script_actor_target(squad)` | `{ code, id, dst, dst_id }` - pursue player (no arrival detection) |
| `is_protected(squad)` | `boolean, reason, detail, name` - check all guards |
| `get_owner(squad)` | `string` or nil - ownership query |
| `register_owner(name, filter_fn)` | register ownership filter (replaces on name match) |
| `register_arrival_handler(key, fn)` | register on-arrival callback by key. `consumer.register` also accepts an `on_arrive` opt that wires this for you |
| `get_scripted_squads()` | read-only reference to `_ap_scripted_squads` table (keyed by squad id; each entry carries `scripted_target`, `target_smart_id`, `tracked_at`, `consequence`, `on_arrive`, `on_arrive_args`, `pre_release_gulag`, `arrived`, `release_at`). The `consequence` field identifies what the squad is doing; `on_arrive` is internal dispatch. |

---

### HUD (ap_core_hud)

Optional hooks for mods that surface their own AP-style markers or want to override the marker subject line.

| Function | Effect |
|----------|--------|
| `register_identity_source(fn)` | add a squad-id list provider. Squads in the returned list receive the `ALPHA_PROMOTE` marker when not currently scripted. Last writer wins on duplicate registration. Today: tracker registers `get_alpha_marker_squads`. |
| `register_subject_resolver(fn)` | replace the marker subject-line resolver (`function(squad) -> string`). Today: `ap_ext_util.format_subject` returns `"<name|species_display> (<faction_display>)"`. |

---

### Const (ap_core_const)

Read-only static tables and enums that integrators reference directly. No registration needed.

| Symbol | Use |
|--------|-----|
| `CAUSE` | enum of cause event names (e.g. `CAUSE.MASSACRE` -> `"cause:massacre"`). Use as xbus event keys. |
| `CONSEQUENCE` | enum of consequence keys (e.g. `CONSEQUENCE.MASSACRE_INVESTIGATE`). Pass to `script_squad` `opts.consequence` for identity; read back from `ap_core_broker.get_scripted_squads()[squad_id].consequence`. Pass to `opts.on_arrive` only if you've registered an arrival handler under that name via `consumer.register`. |
| `CONSEQUENCE_INFO` | per-consequence `{ name_key, action_key }`. `name_key` is the short caption ("Massacre Investigate"); `action_key` is the full action phrase ("Investigating a Massacre Site"). Both XML ids resolved via `game.translate_string`. |
| `CONSEQUENCE_PHASE` | trace-only enum used by `observe()` for sub-phase paths (FIND_TARGETS, MOVE_SQUAD, ARRIVE, etc.). Integrators rarely need this; it shows up in DEBUG-level traces. |
| `RESULT` | consequence handler return codes (SUCCESS, FAILED_RULES, FAILED_SCAN, FAILED_ACTION, DISABLED). |
| `REASON` | reason codes for skip/failure traces (NIL_ARGS, IS_OWNED, NO_SQUAD, ...). |
| `RANGE_EYE` / `RANGE_RADIO` / `RANGE_SCENT` | range tier constants in meters. Use when calling `ap_core_util.find_squads`/`find_smart`. |

`script_squad` does not check protection. Caller must verify:

```lua
local protected = ap_core_broker.is_protected(squad)
if protected then return end
ap_core_broker.script_squad(squad, smart, {
    consequence = CONSEQUENCE.MY_CONSEQUENCE,
    rush = true,
    on_arrive = CONSEQUENCE.MY_CONSEQUENCE,
    on_arrive_args = { target_id = target.id },
    pre_release_gulag = 1800,
})
```

---

### Warfare map tooltip: display AP activity

Goal: when your warfare map UI hovers a squad, append what the squad is doing according to AlifePlus ("Investigating a Massacre Site", "Guarding an Outpost", "Hunting the Wounded"). Strings are localized; both English and Russian ship with AP. Copy-paste example; warfare author edits only the call site at the bottom:

```lua
--- Return localized AP action phrase for a squad, or nil if nothing to show.
--- Graceful no-op when AlifePlus is absent or the squad is not currently scripted by AP.
local function get_ap_action(squad_id)
    if not ap_core_broker or not ap_core_const then return nil end

    local scripted = ap_core_broker.get_scripted_squads()[squad_id]
    if not scripted or not scripted.consequence then return nil end

    local info = ap_core_const.CONSEQUENCE_INFO[scripted.consequence]
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

- Call `get_scripted_squads()` fresh on every render tick. Entries mutate as squads move between consequences.
- `ap_core_const.CONSEQUENCE_INFO` is a static const table; read it directly, no caching needed.
- Each entry has both `action_key` (full phrase shown to the player) and `name_key` (short caption for config / debug). Pick whichever fits your UI.
- Locale switch (English / Russian) works automatically because the resolve happens at render time via `game.translate_string`.
- A nil return means the squad is not currently scripted by AP (or is warfare-owned, since warfare-owned squads are excluded at the protection gate and never enter scripting). Render nothing.

Typical action outputs (EN): "Investigating a Massacre Site", "Scavenging a Massacre Site", "Reinforcing Attacked Base", "Evacuating Attacked Base", "Hunting the Wounded", "Guarding an Outpost", "Out Exploring", "Heading to a Campfire to Rest", "Restocking at Trader", "Harvesting Artefacts". One entry per AP consequence; full map in `ap_core_const.CONSEQUENCE_INFO`.

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
