# AlifePlus Conventions

Rules for writing AlifePlus code. Sits between `doc/standards/code-standards.md` (language-level) and `architecture.md` (system design).

---

## Naming

### Cause-Consequence Rule

```
CONSEQUENCE = CAUSE + VERB
```

| Element | Type | Description |
|---------|------|-------------|
| CAUSE | noun | What happened or was observed (state/event) |
| VERB | verb | The response action |
| CONSEQUENCE | cause_verb | The full reaction |
| ACTION | verb | Atomic operation (tracing only) |

Do NOT confuse causes with consequences:
- Cause = noun. What happened. WOUNDED, MASSACRE, ALPHASPOT.
- Consequence = cause + verb. What to do about it. WOUNDED_HUNT, MASSACRE_SCAVENGE, ALPHASPOT_FLEE.
- Verbs (REINFORCE, FLEE, CONQUER) are consequence suffixes, never causes.
- States (WOUNDED, ALPHA) are causes, never consequences.

### Cause Names

Single word preferred. UPPER_SNAKE in constants.

| Rule | Detail | Examples |
|------|--------|----------|
| Single word | Default, always preferred | MASSACRE, WOUNDED, HARVEST, STASH, AREA, ALPHA |
| Plural for variant | Same subject, different scope | AREA (one) vs AREAS (multiple) |
| Compound + KILL | Death event | BASEKILL, ALPHAKILL, SQUADKILL |
| Compound + SPOT | Perception event | ALPHASPOT |
| Single word | Timer-based human need | HUNGER, SLEEP, JOB |
| Closed suffix set | KILL and SPOT | New suffixes require convention update |

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
| PDA message key | `text_pda_consequence_{name}` | `text_pda_consequence_massacre_scavenge` |
| Script file (cause) | `ap_cause_{name}.script` | `ap_cause_massacre.script` |
| Script file (consequence) | `ap_consequence_{name}.script` | `ap_consequence_massacre_scavenge.script` |
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
| Rate limiter | Producer | Sliding window, checked before predicate |
| Enabled gate | Predicate | MCM toggle, early return |
| Key-cache | Predicate | Deduplication for high-frequency callbacks |
| World-state filter | Predicate | Business logic |
| Level reset | Predicate | Clear state on level change |

### Predicate Order

enabled check -> key-cache -> world-state filter -> build payload

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
2. **Eval** - find_squads, find_smart, xobject.se lookups -> `FAILED_EVAL`
3. **Action** - script_squad, record, PDA message -> `FAILED_ACTION` or `SUCCESS`

Gate order within rules: alignment -> species -> personality -> match -> validation.

### Result Codes

Template phase outcomes. Each code names the phase that answered.

| Code | Phase | Meaning |
|------|-------|---------|
| `SUCCESS` | action | Consequence executed, squad scripted, record written |
| `FAILED_RULES` | rules | Business rules rejected (alignment, personality, validation) |
| `FAILED_EVAL` | eval | World query found nothing (no squads, no smart, entity gone) |
| `FAILED_ACTION` | action | Rules and eval passed but action failed (script_squad rejected) |
| `DISABLED` | consumer | Condition pre-gate returned false (MCM toggle off). Never returned by handlers. |

Rules:
- Handlers return SUCCESS, FAILED_RULES, FAILED_EVAL, or FAILED_ACTION
- DISABLED is consumer-only (condition pre-gate, line consumer:68)
- Consumer continues to next consequence on any non-SUCCESS result
- Consumer stops loop only when global radiant budget is exhausted
- On SUCCESS: consumer increments per-type counter, global radiant counter, sets `event_data._fired = true`

### MCM Fields

| Setting | Type | Default |
|---------|------|---------|
| `consequence_{name}_enabled` | bool | true |
| `consequence_{name}_chance` | 0-100 | 10 |
| `consequence_{name}_pda_chance` | 0-100 | 20 |

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
