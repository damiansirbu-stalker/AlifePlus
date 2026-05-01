# AlifePlus Conventions

Rules for writing AlifePlus code. Sits between `doc/standards/code-standards.md` (language-level) and `architecture.md` (system design).

**Structural rules live in `architecture.md`.** The Cause/Consequence Structural Rules section there governs umbrella files, generator pattern, specific-causes-only, N:M cause-consequence mapping, `_set` naming, CONFIGS factory scope, and toggle requirements. This document covers naming details, signatures, log format, and similar micro-conventions only. If a rule appears in both, architecture.md wins.

---

## Naming

### Cause-Consequence Rule

```
CONSEQUENCE = CAUSE + VERB
```

- **Cause** = perception or trigger event. A noun, optionally with a state qualifier. Never a verb.
- **Consequence** = the action taken in response. Full cause name + action verb.

The underscore is the structural separator between cause and verb in the consequence. Verbs only appear in consequences.

| Element | Type | Description |
|---------|------|-------------|
| CAUSE | noun (+ optional state qualifier) | What was perceived or what triggered |
| VERB | verb | The response action |
| CONSEQUENCE | cause_verb | The full reaction |
| ACTION | verb | Atomic operation inside a handler (tracing only) |

### Cause Names

| Rule | Detail | Examples |
|------|--------|----------|
| Single noun | Default for unique cause nouns. | MASSACRE, WOUNDED, HARVEST, ALPHA |
| Compound noun (closed suffixes) | KILL and SPOT remain conventional compound suffixes for compound noun causes. New suffixes require convention update. | BASEKILL, ALPHAKILL, SQUADKILL, ALPHASPOT |
| State qualifier | Admitted when the cause perceives a specific world state. One underscore between noun and qualifier. | STASH_EMPTY, STASH_FULL, AREA_LAIR |
| Umbrella prefix | Required only when cause nouns would collide between families. Use the family's umbrella name as prefix. | NEED_HUNGER (stalker NEEDS) vs INSTINCT_HUNGER (mutant INSTINCTS) |
| No umbrella prefix | When cause names are unique across families, no prefix needed. | Reactions and opportunities never carry umbrella prefix. |
| No verbs in cause | Verbs are actions; actions belong to consequences. | NOT STASH_LOOT, NOT AREA_CONQUER, NOT INSTINCT_FEED — these are consequences |

### When prefix, when not

| Family | Prefix | Reason |
|--------|--------|--------|
| Reactions | none | unique nouns per world event |
| Opportunities | none | unique nouns per perception, optionally with state qualifier |
| Needs (stalker) | `need_` | drive nouns collide with mutant equivalents |
| Instincts (mutant) | `instinct_` | drive nouns collide with stalker equivalents |

### Consequence Names

| Rule | Detail | Examples |
|------|--------|----------|
| Cause + action verb | Consequence appends an action verb to the full cause name. One additional underscore. | MASSACRE_INVESTIGATE, STASH_FULL_LOOT, NEED_HUNGER_CAMPFIRE |
| Forks | When multiple consequences subscribe to the same cause, each has a distinct action verb. | NEED_SHELTER_INDOOR, NEED_SHELTER_OUTDOOR |
| Always carries verb | Even at 1:1 cause-consequence mapping, the consequence has an explicit action verb. | MASSACRE_INVESTIGATE, never bare MASSACRE as consequence |

### All Naming Patterns

| Category | Pattern | Examples |
|----------|---------|----------|
| Cause const | `CAUSE.{NAME}` | `CAUSE.MASSACRE`, `CAUSE.ALPHAKILL` |
| Cause value | `"cause:{name}"` | `"cause:massacre"`, `"cause:alphakill"` |
| Consequence const | `CONSEQUENCE.{CAUSE}_{VERB}` | `CONSEQUENCE.MASSACRE_SCAVENGE` |
| Consequence value | `"consequence:{cause}_{verb}"` | `"consequence:massacre_scavenge"` |
| Action ID | `action:{verb}` | `action:find_targets`, `action:move_squad` |
| Lock (cause) | `lock:cause:{name}` | `lock:cause:massacre` |
| Lock (consequence) | `lock:consequence:{name}` | `lock:consequence:massacre_scavenge` |
| Lock (hit mod) | `lock:hit_modifier` | |
| MCM cause enabled | `cause_{name}_enabled` | `cause_massacre_enabled` |
| MCM cause setting | `cause_{name}_{setting}` | `cause_massacre_threshold` |
| MCM consequence enabled | `consequence_{name}_enabled` | `consequence_massacre_scavenge_enabled` |
| MCM consequence setting | `consequence_{name}_{setting}` | `consequence_massacre_scavenge_chance` |
| MCM distributor | `distributor_{setting}` | `distributor_max_xray_events` |
| MCM cause window | `cause_window_{setting}` | `cause_window_max_events` |
| MCM consequence window | `consequence_window_{setting}` | `consequence_window_max_events` |
| Script file (cause) | `ap_ext_cause_{family}.script` | `ap_ext_cause_massacre.script`, `ap_ext_cause_area.script` (umbrella) |
| Script file (consequence, single) | `ap_ext_consequence_{name}.script` | `ap_ext_consequence_massacre_scavenge.script` |
| Script file (consequence, umbrella) | `ap_ext_consequence_{family}_set.script` | `ap_ext_consequence_area_set.script` |
| Community list | `community_{role}` | `community_stalker`, `community_predator` |
| Log prefix (cause) | `CAUSE.{NAME}` | `CAUSE.MASSACRE` |
| Log prefix (consequence) | `CONSEQUENCE.{NAME}` | `CONSEQUENCE.MASSACRE_SCAVENGE` |
| MCM menu ID | `{name}` (lowercase) | `alpha_promote`, `massacre_scavenge` |
| MCM sidebar | 1 word per line (underscore = newline) | `Alpha\nKill\nTargeted` |
| XML title (cause) | `ui_mcm_ap_causes_{name}_title` | `ui_mcm_ap_causes_massacre_title` |
| XML title (consequence) | `ui_mcm_ap_consequences_{name}_title` | `ui_mcm_ap_consequences_massacre_scavenge_title` |
| Chase handler key | `{cause}_chase` | `squadkill_chase`, `alphakill_chase` |
| Arrival handler key | consequence name (1:1) | `stash_loot`, `stash_fill` |
| DTO table | `stalker_needs[squad_id]` | future: `mutant_needs[squad_id]` |
| DTO field | `last_{short}_at` | `last_hunger_at`, `last_sleep_at` |

---

## Cause Standard Pattern

Causes are predicates. Return `{ cause = CAUSE.X, ...payload }` or `nil`.

### Signature

`function(trace, ...callback_args) -> { cause = CAUSE.X, ...payload } | nil`

### Ownership

| Element | Owner | Purpose |
|---------|-------|---------|
| Rate limiter | Producer | Per-CATEGORY sliding window, checked pre-handler in `_eval_*` |
| Enabled gate | Predicate | MCM toggle (`cause_<name>_enabled`), early return |
| World-state filter | Predicate | Business logic that decides whether to publish |

### Predicate Order

enabled check -> world-state filter -> build payload -> self-observe (prof+trace:push+debug)

Each predicate publishes exactly one specific cause. Umbrella cause files (`ap_ext_cause_<family>.script`) hold one predicate per cause; each predicate is independent.

### MCM Fields

| Setting | Type | Default |
|---------|------|---------|
| `cause_{name}_enabled` | bool | true |

---

## Consequence Standard Pattern

Consequence handlers receive trace from consumer. Rate limiting handled by consumer.

### Signature

`function(event_data) -> { code = RESULT.X, reason = "..." }`

### Ownership

| Element | Owner | Purpose |
|---------|-------|---------|
| Rate limiter | Consumer | Token bucket (per-consequence) + global counter (radiant), checked before handler |
| Enabled gate | Consumer | MCM condition function, checked before handler |
| Rules phase | Handler | Alignment, species, personality, validation |
| Eval phase | Handler | find_squads, find_smart, entity lookups |
| Action phase | Handler | script_squad, record, PDA message |
| Result code | Handler | Template phase outcome |

### Handler Template (rules -> eval -> action)

Every consequence follows a three-phase structure. Each phase returns immediately on failure.

1. **Rules** - alignment check, species check (direct hash), personality roll, match validation, at_base guard -> `FAILED_RULES`
2. **Eval** - find_squads, find_smart, xobject.se lookups -> `FAILED_SCAN`
3. **Action** - script_squad, record, PDA message -> `FAILED_ACTION` or `SUCCESS`

Gate order within rules: alignment -> species -> personality -> match -> validation.

### Result Codes

Template phase outcomes. Each code names the phase that answered.

| Code | Phase | Meaning |
|------|-------|---------|
| `SUCCESS` | action | Consequence executed, squad scripted, record written |
| `FAILED_RULES` | rules | Business rules rejected (alignment, personality, validation) |
| `FAILED_SCAN` | eval | World query found nothing (no squads, no smart, entity gone) |
| `FAILED_ACTION` | action | Rules and eval passed but action failed (script_squad rejected) |
| `DISABLED` | consumer | Condition pre-gate returned false (MCM toggle off). Never returned by handlers. |

Rules:
- Handlers return SUCCESS, FAILED_RULES, FAILED_SCAN, or FAILED_ACTION
- DISABLED is consumer-only (condition pre-gate, line consumer:68)
- Consumer continues to next consequence on any non-SUCCESS result
- Consumer stops loop only when global radiant budget is exhausted
- On SUCCESS: consumer increments per-type counter, global radiant counter, sets `event_data._fired = true`

### MCM Fields

| Setting | Type | Default |
|---------|------|---------|
| `consequence_{name}_enabled` | bool | true |
| `consequence_{name}_chance` | 0-100 | 10 |

### Development

| Setting | Type | Default | Controls |
|---------|------|---------|----------|
| `log_level` | enum | WARN | Log verbosity (ERROR/WARN/INFO/DEBUG) |

`log_level = DEBUG` enables tracing (xtrace), performance timing (xprofiler), and PDA map markers. When `log_level < DEBUG`, timers return null singletons (zero overhead), markers suppressed via `ap_debug.enabled()` gate.

---

## A-Life Rules

- **ID-based** - Track by server ID, not game_object (ephemeral)
- **Squad-based** - Squads are atomic units, NPCs are members
- **Bias not command** - scripted_target is suggestion, not force
- **Same-level** - Operations constrained to current level
- **No spawning** - Redirect existing entities only

---

## Logging

### Format

```
[{component}] [traceId={id} path={path}] {message} {key=value pairs}
```

### Component Prefixes

| Prefix | Scope |
|--------|-------|
| `CONSEQUENCE.{NAME}` | Consequence |
| `CAUSE.{NAME}` | Cause handler |
| `PRODUCER.REACTIVE` | Producer dispatch |
| `DISPATCH` | Event publish (ap_utils) |
| `CONSUMER` | Cause consumer routing |
| `MUTATOR.OBJECT` | Runtime combat modifiers (ap_object_mutator) |
| `MUTATOR.SMART` | Territory conquest (ap_smart_mutator) |
| `TEST` | MCM test tools (ap_test) |

### Tracing

| Term | Definition |
|------|------------|
| `traceId` | Unique ID for a trace (monotonic counter) |
| `path` | Breadcrumb trail like `massacre(5)/loot(6)` |

No spanId - path contains operation names with counters.

### Log Levels

| Level | Purpose | Examples |
|-------|---------|----------|
| ERROR | pcall failures, real errors | Handler failed |
| WARN | Severe issues, degraded state | Failed to give item |
| INFO | Startup, save/load summaries | Initialized, SAVE: N killers |
| DEBUG | Everything else | Events, traces, timing, skips |

### Rules

| Rule | Detail |
|------|--------|
| observe() location | Lives in `ap_debug`. Guarded by `ap_debug.enabled()`. When debug off, straight pass-through with zero overhead. |
| Generic serializer | Inside `observe()`. Iterates result table pairs, logs all scalar fields as `k=v`. Skips userdata, table, function, nil. Skips `code`/`reason`. |
| Result builders | `ap_debug.result_squads(squads, extra)`, `ap_debug.result_squad(squad, extra)`. Never manually collect IDs. |
| No engine calls for logging | Log only what's already computed. IDs over names. Never fetch names just for logging. |
| Lazy name cache | Per-method in AlifePlus code. Uses xlib calls, never raw xray/luabind. |
| Action closure returns | All closures must return standardized tables using result builders. |
| Cause returns | Predicates return `{ cause, ...payload }` with all scalars auto-logged by generic serializer. |
| Consequence outer returns | Enriched with free scalars: `count`, `ids`, `dst_id`, `faction` via result builders. |
| Trace goal | Grep a `tid` -> see full cause->consequence->action chain with all linking IDs. |

---

## Critical Rules

### Level is the Level of the Event

All events MUST include `level_id` from where the event occurred.

**Cause responsibility:**
- Extract `level_id` from the entity using `xworld.get_level_id(se_obj)`
- Include `level_id` in the published event payload

**Consequence responsibility:**
- Use `event_data.level_id` for all location-based operations
- Pass `level_id` to `ap_ext_news.record_event(squad, cause, consequence, { level_id = ... })` so the journal captures it

Never use `get_actor_level_id()` as fallback. The player may be on a different level.
