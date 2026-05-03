# AlifePlus Conventions

Rules for writing AlifePlus code. Sits between `doc/standards/code-standards.md` (language-level) and `architecture.md` (system design).

**Structural rules live in `architecture.md`.** The Cause/Consequence Structural Rules section there governs umbrella files, generator pattern, specific-causes-only, N:M cause-consequence mapping, `_set` naming, CONFIGS factory scope, and toggle requirements. This document covers naming details, signatures, log format, and similar micro-conventions only. If a rule appears in both, architecture.md wins.

---

## Naming

The naming rule splits by pipeline. Radiant pairs are 1:1 and share the noun. Reactive causes are 1:N and keep the cause-noun + consequence-verb pattern.

### Radiant: cause and consequence share the noun (invariant 10)

```
cause:<noun>  ↔  consequence:<noun>
```

Per the radiant rule that cause and consequence share the noun (see `architecture.md` Rules). The 1:1 bond is visible in the name. No invented synonyms to make a "cause noun" sound different from a "consequence verb" — they are the same concept under two scopes (the trigger state and the action).

| Family | Pattern | Examples |
|--------|---------|----------|
| Opportunities (area) | `area_<noun>` shared on both sides | `cause:area_conquer` ↔ `consequence:area_conquer`; `cause:area_swarm` ↔ `consequence:area_swarm`; `cause:area_infest` ↔ `consequence:area_infest` |
| Opportunities (stash) | `stash_<noun>` shared on both sides | `cause:stash_loot` ↔ `consequence:stash_loot`; `cause:stash_ambush` ↔ `consequence:stash_ambush`; `cause:stash_fill` ↔ `consequence:stash_fill` |
| Needs (stalker) | `<drive>_<answer>` shared on both sides | `cause:hunger_campfire` ↔ `consequence:hunger_campfire`; `cause:shelter_indoor` ↔ `consequence:shelter_indoor`; `cause:job_outpost` ↔ `consequence:job_outpost` |
| Instincts (mutant) | bare drive noun for single-answer drives; `<drive>_<answer>` for multi-answer drives | single-answer: `cause:feed` ↔ `consequence:feed`; `cause:scatter` ↔ `consequence:scatter`. Multi-answer (slumber): `cause:slumber_field` ↔ `consequence:slumber_field`; `cause:slumber_lair` ↔ `consequence:slumber_lair`; `cause:slumber_surge` ↔ `consequence:slumber_surge`. |

Family prefix presence depends on collision: needs and instincts use no family prefix because their drive nouns do not overlap (stalker drives are hunger, sleep, rest, heal, shelter, supply, money, job, social; mutant drives are scatter, feed, slumber, roam, pack). Stash and area use family prefixes for grouping.

### Reactive: cause-noun + consequence-verb (1:N)

Reactive causes can fan out to multiple consequences (1:N is allowed for reactive per concept). Each consequence carries its own action verb appended to the cause name.

```
CONSEQUENCE = CAUSE + VERB
```

| Rule | Detail | Examples |
|------|--------|----------|
| Cause noun | Single noun describing the world event | MASSACRE, WOUNDED, HARVEST, ALPHA |
| Compound noun (closed suffixes) | KILL and SPOT remain conventional compound suffixes for compound noun causes | BASEKILL, ALPHAKILL, SQUADKILL, ALPHASPOT |
| Consequence pattern | Full cause name + action verb, single underscore separator | MASSACRE_INVESTIGATE, MASSACRE_SCAVENGE, BASEKILL_FLEE |
| Forks | Multiple consequences on the same reactive cause each carry a distinct verb | WOUNDED_HUNT, WOUNDED_HELP |

### All Naming Patterns

| Category | Pattern | Examples |
|----------|---------|----------|
| Cause const | `CAUSE.{NAME}` | `CAUSE.MASSACRE`, `CAUSE.AREA_CONQUER`, `CAUSE.HUNGER_CAMPFIRE`, `CAUSE.SLUMBER_LAIR` |
| Cause value | `"cause:{name}"` | `"cause:massacre"`, `"cause:area_conquer"`, `"cause:hunger_campfire"`, `"cause:slumber_lair"` |
| Consequence const (radiant) | `CONSEQUENCE.{NAME}` — same noun as the cause | `CONSEQUENCE.AREA_CONQUER`, `CONSEQUENCE.HUNGER_CAMPFIRE`, `CONSEQUENCE.SLUMBER_LAIR` |
| Consequence const (reactive) | `CONSEQUENCE.{CAUSE}_{VERB}` | `CONSEQUENCE.MASSACRE_SCAVENGE`, `CONSEQUENCE.WOUNDED_HUNT` |
| Consequence value (radiant) | `"consequence:{name}"` — same noun as the cause | `"consequence:area_conquer"`, `"consequence:hunger_campfire"`, `"consequence:slumber_lair"` |
| Consequence value (reactive) | `"consequence:{cause}_{verb}"` | `"consequence:massacre_scavenge"` |
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
| Script file (cause, single) | `ap_ext_cause_{family}.script` — generator publishes exactly one cause | `ap_ext_cause_massacre.script`, `ap_ext_cause_alpha.script` |
| Script file (cause, multi) | `ap_ext_causes_{family}.script` — generator publishes multiple causes | `ap_ext_causes_area.script`, `ap_ext_causes_stash.script`, `ap_ext_causes_needs.script`, `ap_ext_causes_instincts.script` |
| Script file (consequence) | `ap_ext_consequences_{family}.script` — holds every consequence subscribed to causes in the family. Internal shape (CONFIGS factory or hand-written N) doesn't affect the file name. | `ap_ext_consequences_alpha.script` (1 handler), `ap_ext_consequences_wounded.script` (2 handlers), `ap_ext_consequences_needs.script` (14 handlers) |
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

## Multi-answer drive (radiant generator pattern)

Each answer is a separate first-class cause paired 1:1 with its own consequence (sharing the noun per the radiant naming rule). "Multi-answer drive" is a code-structure quirk: a Hull-scored drive can be satisfied by any of several answers depending on squad identity (faction, species). The drive itself owns only the Hull threshold and the DTO timestamp field; everything else (enable, alignment, personality traits, filter) belongs to the individual answers. Any answer firing resets the drive's timestamp.

Two tables in the generator file:

- **NEEDS / INSTINCTS** (one entry per drive) — Hull weight, threshold cfg key, period gating, DTO field name.
- **CAUSES** (one entry per answer) — cause const, short name (cfg key suffix), parent drive name, alignment subset, personality, filter, optional `find_opts` builder.

Picker flow: score every drive via Hull, sort overdue list by drive descending, walk overdue drives top-down. For each drive, walk CAUSES with matching parent drive, run RULES + SCAN per cause; first that publishes wins. Cap is enforced by the producer (RADIANT_MAX_CHECKS_PER_TICK).

cfg key layout:
- `cause_<drive>_threshold` — Hull threshold, one cfg key per drive (shared by every answer under that drive)
- `cause_<answer>_enabled` — per-cause enable, one cfg key per answer. For single-answer drives the answer name equals the drive name (`feed`, `roam`, `pack`, `scatter`), so the cfg key reads as `cause_<drive>_enabled` but it is conceptually per-answer.
- `consequence_<answer>_enabled` — consequence enable
- `consequence_<answer>_rush` — rush option

Personality clamp is global, not per-cause. `PERSONALITY_FLOOR` / `PERSONALITY_CEILING` in `ap_ext_const` apply uniformly to every cause and consequence personality roll. No per-cause min/max cfg keys.

When to use: a drive that has multiple alternative satisfactions (mutant slumber → field/lair/surge by species; future fear drive → flee/hide/freeze). Don't use for single-answer drives where the answer name would equal the drive name (`scatter` is one answer to scatter drive — no split needed; cfg keys collapse).

When NOT to use: state classifiers where the picker selects exactly one branch by inspecting world state (stash empty/full/trap pattern). Those use a state-by-state KEYS_BY_CAUSE picker, not Hull cascade.

Used by: `ap_ext_cause_needs.script` (9 drives, 14 answers), `ap_ext_cause_instincts.script` (5 drives, 7 answers, multi-answer slumber).

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
| Scan phase | Handler | find_squads, find_smart, entity lookups |
| Action phase | Handler | script_squad, record, PDA message |
| Result code | Handler | Template phase outcome |

### Handler Template (rules -> scan -> action)

Every consequence follows a three-phase structure. Each phase returns immediately on failure.

1. **Rules** - alignment check, species check (direct hash), personality roll, match validation, at_base guard -> `FAILED_RULES`
2. **Scan** - find_squads, find_smart, xobject.se lookups -> `FAILED_SCAN`
3. **Action** - script_squad, record, PDA message -> `FAILED_ACTION` or `SUCCESS`

Gate order within rules: alignment -> species -> personality -> match -> validation.

### Result Codes

Template phase outcomes. Each code names the phase that answered.

| Code | Phase | Meaning |
|------|-------|---------|
| `SUCCESS` | action | Consequence executed, squad scripted, record written |
| `FAILED_RULES` | rules | Business rules rejected (alignment, personality, validation) |
| `FAILED_SCAN` | scan | World query found nothing (no squads, no smart, entity gone) |
| `FAILED_ACTION` | action | Rules and scan passed but action failed (script_squad rejected) |
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
- **Synchronous payload only** - Server userdata in cause payloads is valid only for the synchronous dispatch chain (producer -> consumer -> consequence handler -> record_event). Anything invoked on a later tick (xlice/CreateTimeEvent jobs, on_arrive callbacks, deferred scanners) references entities by server ID and re-resolves via `xobject.se(id)` / `alife_object(id)`.
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

---

## MCM

### Tag widget

Each MCM cause section uses one tag widget (`_tag()`, faded khaki) to display three lines: identity, description, rules. No separate `_desc()` widget — the description is folded into the tag so it shares the faded style with the rest.

### Line order inside the tag

```
Type: <cause-type>, <mechanic>          ← umbrella (family-level) section
   OR
Cause: <Display Name>                    ← per-specific-cause section
Desc: <one-line description>
Rules: <semicolon-separated clauses>
```

### Rules clause order

Semicolon-separated, always in this order:

1. `alignment_<set>`
2. `personality(<TRAIT>, <TRAIT>)` — adjacent to alignment
3. world-state predicate (`not at base`, `stash non-empty`, `items >= min`, etc.)
4. threshold mechanism (`Hull(<drive>)` or `MVT(<cause>)`)

Alignment and personality always sit adjacent at the front. Branch-specific clauses (state, threshold) follow.

### Display name vs code identifier

The `Cause:` line uses display-name form: title-case, space-separated (`Stash Loot`, `Area Conquer`, `Hunger Campfire`). Never the cfg-key segment (`stash_loot`).

For radiant causes only, the display name carries the suffix ` Available` on the `Cause:` line (e.g. `Cause: Stash Loot Available`, `Cause: Hunger Campfire Available`). This disambiguates the cause from its 1:1 consequence which shares the same noun internally per the radiant rule that cause and consequence share the noun. Reactive causes do not need the suffix because their causes and consequences already differ in name (massacre vs massacre_investigate, etc.). The consequence display name stays bare (`Stash Loot`, not `Stash Loot Available`).

`MVT(<cause>)`, `Hull(<drive>)`, `personality(<trait>, <trait>)` are formal architecture vocabulary used in MCM tags as-is. Cause/drive/trait names are lowercase. Code enum constants (`CAUSE.STASH_LOOT`, `PERSONALITY.TERRITORY`) stay uppercase as Lua identifiers — formal vocabulary references the lowercase string form.

`alignment_<set>` is internal vocabulary for technical docs (architecture.md, this file, integration-guide.md) only. User-facing MCM text uses plain English instead:

| Code identifier | Plain English (user-facing MCM) |
|---|---|
| `alignment_human` | `any stalker` |
| `alignment_loot` | `unprincipled or outlaw stalkers` |
| `alignment_outlaw` | `outlaw stalkers` |
| `alignment_conquer_human` | `any stalker except ecologists` |
| `alignment_conquer_mutant` | `feral, predator, or aberrant mutants` |
| `alignment_mutant` | `any mutant` |
| `alignment_principled` | `principled stalkers` |
| `alignment_selfserving` | `self-serving stalkers` |
| `alignment_unprincipled` | `unprincipled stalkers` |
| `alignment_naturalist` | `naturalist stalkers` |
| `alignment_ecolog` | `ecologists` |
| `alignment_mutant_cowardly` | `cowardly mutants` |
| `alignment_mutant_feral` | `feral mutants` |
| `alignment_mutant_predator` | `predator mutants` |
| `alignment_mutant_aberrant` | `aberrant mutants` |

### Section layout

Family with multiple specific causes (e.g. stash, area, needs, instincts): one umbrella tag at the top (`Type:` line), then for each specific cause: a line divider followed by the sub-cause tag (`Cause:` line) and the cause's settings (enable toggle, threshold, personality min/max, any branch-only sliders).

Family with a single cause (e.g. alpha, alphakill, wounded): just one tag, no sub-divisions.
