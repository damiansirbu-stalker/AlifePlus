# AlifePlus Conventions

Rules for writing AlifePlus code. Sits between `doc/standards/code-standards.md` (language-level) and `architecture.md` (system design).

**Structural rules live in `architecture.md`.** The Rules section there governs umbrella files, generator pattern, specific-causes-only, cause-consequence mapping, `_set` naming, CONFIGS factory scope, and toggle requirements. This document covers naming details, signatures, log format, and similar micro-conventions only. If a rule appears in both, architecture.md wins.

---

## Naming

The naming rule splits by pipeline. Radiant pairs are 1:1 and share the noun. Reactive causes are 1:N and keep the cause-noun + consequence-verb pattern.

### Radiant: cause and consequence share the noun (invariant 10)

```
cause:<noun>  ↔  consequence:<noun>
```

Per the radiant rule that cause and consequence share the noun (see `architecture.md` Rules). The 1:1 bond is visible in the name. No invented synonyms to make a "cause noun" sound different from a "consequence verb"; they are the same concept under two scopes (the trigger state and the action).

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
| Consequence const (radiant) | `CONSEQUENCE.{NAME}`; same noun as the cause | `CONSEQUENCE.AREA_CONQUER`, `CONSEQUENCE.HUNGER_CAMPFIRE`, `CONSEQUENCE.SLUMBER_LAIR` |
| Consequence const (reactive) | `CONSEQUENCE.{CAUSE}_{VERB}` | `CONSEQUENCE.MASSACRE_SCAVENGE`, `CONSEQUENCE.WOUNDED_HUNT` |
| Consequence value (radiant) | `"consequence:{name}"`; same noun as the cause | `"consequence:area_conquer"`, `"consequence:hunger_campfire"`, `"consequence:slumber_lair"` |
| Consequence value (reactive) | `"consequence:{cause}_{verb}"` | `"consequence:massacre_scavenge"` |
| Action ID | `action:{verb}` | `action:find_targets`, `action:move_squad` |
| MCM cause enabled | `cause_{name}_enabled` | `cause_massacre_enabled` |
| MCM cause setting | `cause_{name}_{setting}` | `cause_massacre_threshold` |
| MCM consequence enabled | `consequence_{name}_enabled` | `consequence_massacre_scavenge_enabled` |
| MCM consequence setting | `consequence_{name}_{setting}` | `consequence_supply_trader_rush` |
| MCM mutator setting | `mutator_{name}_{setting}` | `mutator_area_conquest_spawn_num`, `mutator_area_infest_decay_hours` |
| Script file (cause, single) | `ap_ext_cause_{family}.script`; generator publishes exactly one cause | `ap_ext_cause_massacre.script`, `ap_ext_cause_alpha.script` |
| Script file (cause, multi) | `ap_ext_causes_{family}.script`; generator publishes multiple causes | `ap_ext_causes_area.script`, `ap_ext_causes_stash.script`, `ap_ext_causes_needs.script`, `ap_ext_causes_instincts.script` |
| Script file (consequence) | `ap_ext_consequences_{family}.script`; holds every consequence subscribed to causes in the family. Internal shape (CONFIGS factory or hand-written `_set`) doesn't affect the file name. | `ap_ext_consequences_alpha.script`, `ap_ext_consequences_needs.script` |
| Log prefix (cause) | `CAUSE.{NAME}` | `CAUSE.MASSACRE` |
| Log prefix (consequence) | `CONSEQUENCE.{NAME}` | `CONSEQUENCE.MASSACRE_SCAVENGE` |
| MCM menu ID | `{name}` (lowercase) | `alpha_promote`, `massacre_scavenge` |
| MCM sidebar | 1 word per line (underscore = newline) | `Alpha\nKill\nTargeted` |
| XML title (cause) | `ui_mcm_ap_causes_{name}_title` | `ui_mcm_ap_causes_massacre_title` |
| XML title (consequence) | `ui_mcm_ap_consequences_{name}_title` | `ui_mcm_ap_consequences_massacre_scavenge_title` |
| On-arrive handler key | consequence name | `stash_loot`, `squadkill_revenge` (radiant arrivals and chase recursion both use the consequence enum string) |
| DTO table | `_ap_{owner}_{domain}[squad_id]` | `_ap_stalker_needs`, `_ap_mutant_instincts`, `_ap_squad_opportunities` (ap_ext_tracker) |
| DTO field | `last_{short}_at` | `last_hunger_at`, `last_sleep_at` |

---

## Multi-answer drive (radiant generator pattern)

Canonical description in architecture.md -> Causes -> Multi-answer drive (tables, picker flow, cfg key layout). Only the cfg-key consequence pair is a naming convention: `consequence_<answer>_enabled` and `consequence_<answer>_rush`, one per answer.

---

## Cause Standard Pattern

Causes are predicates. Return `{ cause = CAUSE.X, ...payload }` or `nil`.

### Signature

`function(trace, ...callback_args) -> { cause = CAUSE.X, ...payload } | nil`

### Ownership

| Element | Owner | Purpose |
|---------|-------|---------|
| Rate limiter | Predicate | Self-gates the per-CAUSE_CATEGORY sliding window (`ap_core_limiter.check_cause_rate_limit` at the top, `increment_cause_counter` on publish); the producer does not rate-check |
| Enabled gate | Predicate | MCM toggle (`cause_<name>_enabled`), early return |
| World-state filter | Predicate | Business logic that decides whether to publish |

### Predicate Order

enabled check -> rate self-gate -> world-state filter -> build payload. The producer times the predicate inline (`xprofiler.new_if(dbg)`) and logs its outcome line; the predicate does not log its own timing.

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

Each code names the template phase that answered: SUCCESS (action), FAILED_RULES (rules), FAILED_SCAN (scan), FAILED_ACTION (action), plus DISABLED, the consumer condition pre-gate skip (never a handler return). Canonical definitions and dispatch semantics: architecture.md -> Dispatch Pipeline -> Per-handler dispatch.

### MCM Fields

| Setting | Type | Default |
|---------|------|---------|
| `consequence_{name}_enabled` | bool | true |

### Development

| Setting | Type | Default | Controls |
|---------|------|---------|----------|
| `log_level` | enum | WARN | Log verbosity (ERROR/WARN/INFO/DEBUG) |

`log_level = DEBUG` enables tracing (xtrace) and performance timing (xprofiler). When `log_level < DEBUG`, timers return null singletons (zero overhead). PDA map markers are independent of log level: they gate on the MCM `map_markers` toggle only (`ap_core_map._update_markers`).

---

## A-Life Rules

- **ID-based** - Track by server ID, not game_object (ephemeral)
- **Synchronous payload only** - Server userdata in cause payloads is valid only for the synchronous dispatch chain (producer -> consumer -> consequence handler -> `ap_core_record.add_record`). Anything invoked on a later tick (xlice/CreateTimeEvent jobs, on_arrive callbacks, deferred scanners) references entities by server ID and re-resolves via `xobject.se(id)` / `alife_object(id)`.
- **Squad-based** - Squads are atomic units, NPCs are members
- **Bias not command** - scripted_target is suggestion, not force
- **Same-level** - Operations constrained to current level
- **No spawning** - Redirect existing entities only

---

## Logging

### Format

```
[{component}] [tid={id}] {message} [X.XXms]
```

The `[tid={id}]` and the `[X.XXms]` duration appear together on a flow line; a line that carries a tid also carries its duration (validator rule `alifeplus-debug-log-duration`). Per-candidate cause lines and plain diagnostic lines carry neither.

### Component Prefixes

Pipeline labels are composed by `ap_core_debug.bracket(constant)` (uppercase, `:` -> `.`): `[CAUSE.{NAME}]`, `[CONSEQUENCE.{NAME}]`, plus the family roots (`[NEEDS]`, `[STASH]`, ...). Non-pipeline modules carry a literal `LOG` prefix: `[EXT.MARKET]`, `[EXT.LOOTSEL]`, `[CORE.CACHE]`, `[SQUAD]`, `[PRODUCER]`. No hardcoded `[CAUSE.X]` literals in pipeline files; each caches its bracket strings at module load (`entry.log_prefix`).

### Tracing

Correlation is a plain integer `tid` (a monotonic counter from `xtrace.new().id`), carried on the payload as `result.tid`. There is no path and no span hierarchy. The producer writes the tid on the cause result; the consumer (`_dispatch_entry`) reads `event_data.tid` and logs each consequence outcome under the same tid. Deferred arrivals mint a fresh tid and correlate by `dst=` / `sq=`, not by tid (the counter resets on VM reinit, so a stored tid would collide after a save). Per-candidate cause lines inside a radiant family are tid-less; the producer's family outcome line carries the tid.

### Scan tracing (`find_smart`, `find_squads`, `find_first_smart`)

The finders are plain (no `*_observed` variant). `ap_core_util.find_squads` / `find_smart` emit a `[DISPATCH.FIND_*]` DEBUG line (count and params, or a no-smart line on miss) under `ap_core_debug.enabled()`; below DEBUG they run straight through with no line. The calling flow's own inline timer owns the timing; the finder pushes no span.

### Log Levels

| Level | Purpose | Examples |
|-------|---------|----------|
| ERROR | pcall failures, real errors | Handler failed |
| WARN | Severe issues, degraded state | Failed to give item |
| INFO | Startup, save/load summaries | Initialized, SAVE: N killers |
| DEBUG | Everything else | Events, flow timing, skips |

### Rules

| Rule | Detail |
|------|--------|
| Inline timing | Each big flow times itself with `xprofiler.new_if(dbg)` and logs one summary line. No per-call wrapper. |
| Timer null below DEBUG | `xprofiler.new_if(false)` returns the shared null singleton (zero allocation, `get_ms()` returns 0); the tid is not minted. |
| No engine calls for logging | Log only what's already computed. IDs over names. Never fetch names just for logging. Gate a DEBUG log whose args read userdata (e.g. `squad:section_name()`) behind `if ap_core_debug.enabled()`. |
| code / reason | `code` and `reason` stay on result tables as control flow and HUD stats, not logging cruft. No other free scalars on result tables. |
| Cause returns | Predicates return `{ cause, ...payload }` on success, `{ code, reason }` on rejection. |
| Trace goal | Grep a `tid` -> see the cause line and its synchronous consequence line. |

---

## Critical Rules

### Level is the Level of the Event

All events MUST include `level_id` from where the event occurred.

**Cause responsibility:**
- Extract `level_id` from the entity using `xlevel.get_level_id(se_obj)`
- Include `level_id` in the published event payload

**Consequence responsibility:**
- Use `event_data.level_id` for all location-based operations
- Pass `level_id` to `ap_core_record.add_record(squad, cause, consequence, { level_id = ... })` so the activity record captures it

Never use `get_actor_level_id()` as fallback. The player may be on a different level.

---

## MCM

### Tag widget

Each MCM cause section uses one tag widget (`_tag()`, faded khaki) per per-cause block. No umbrella tag at the family level; multi-cause families (area, stash, needs, instincts) render each sub-cause as a self-contained tag with a divider between them. The slide banner at the top of the page provides the family identity.

### Line order inside the tag

```
Cause: <Display Name> Available    ← radiant; "Available" suffix only on radiant
Cause: <Display Name>              ← reactive consequence
Desc: <one short sentence; no implementation jargon>
Range: <eye | radio | scent>
Period: <active | dormant>         ← radiant only; reactions omit
Rules: <semicolon-separated clauses, max 3 words each>
```

Range and Period are intrinsic properties of the cause, separated from filter rules. Reactions omit `Period:` (they fire on engine events, not gated by squad period).

### Rules clause order

Semicolon-separated, always in this order:

1. `alignment_<set>`
2. `personality(<TRAIT>, <TRAIT>)`; adjacent to alignment
3. world-state predicate (`not at base`, `stash non-empty`, `items >= min`, etc.)
4. threshold mechanism (`Hull(<drive>)` or `MVT(<cause>)`)

Alignment and personality always sit adjacent at the front. Branch-specific clauses (state, threshold) follow.

### Clause brevity

Max 3 words per clause. Hyphenated compounds count as one word; formal vocabulary forms like `personality(territory, aggression)`, `Hull(<drive>)`, `MVT(<cause>)` count as one token regardless of inner length. Union alignments (multi-set merges) use a single short label (`non-cowardly mutants`, `non-ecologist stalkers`) rather than enumerating the members. If a clause naturally needs more than 3 words, coin a shorter label or split it into two clauses.

### Display name vs code identifier

The `Cause:` line uses display-name form: title-case, space-separated (`Stash Loot`, `Area Conquer`, `Hunger Campfire`). Never the cfg-key segment (`stash_loot`).

For radiant causes only, the display name carries the suffix ` Available` on the `Cause:` line (e.g. `Cause: Stash Loot Available`, `Cause: Hunger Campfire Available`). This disambiguates the cause from its 1:1 consequence which shares the same noun internally per the radiant rule that cause and consequence share the noun. Reactive causes do not need the suffix because their causes and consequences already differ in name (massacre vs massacre_investigate, etc.). The consequence display name stays bare (`Stash Loot`, not `Stash Loot Available`).

`MVT`, `Hull`, `personality(<trait>, <trait>)` are formal architecture vocabulary used in MCM tags. Inside a per-cause section the cause/drive name is already clear from context, so `MVT` and `Hull` are written bare; no parenthesized inner name. Personality keeps its traits because they vary across causes in the same family. When a parenthesized inner name IS needed (multi-answer drive header that names the shared drive, or cross-section reference), use the display-name form: `MVT(Area Conquer)` not `MVT(area_conquer)`. Single-word inner names that read as ordinary English (`Hull(hunger)`, `personality(territory, aggression)`) stay lowercase. Code enum constants (`CAUSE.STASH_LOOT`, `PERSONALITY.TERRITORY`) stay uppercase as Lua identifiers.

### No identifiers in MCM

Never put underscore-joined programmer identifiers (`area_conquer`, `stash_loot`, `cause_area_infest_threshold`) in user-facing MCM text; slider labels, descriptions, tag bodies. Use the display-name form (`Area Conquer`, `Stash Loot`) inside formal vocabulary, and plain English everywhere else. The cfg-key (`cause_area_conquer_threshold`) lives in code only; the user sees `MVT(Area Conquer)`.

`alignment_<set>` is internal vocabulary for technical docs (architecture.md, this file, integration.md) only. User-facing MCM text uses plain English instead:

| Code identifier | Plain English (user-facing MCM) |
|---|---|
| `alignment_human` | `any stalker` |
| `alignment_loot` | `loot-seeking stalkers` |
| `alignment_outlaw` | `outlaw stalkers` |
| `alignment_conquer_human` | `non-ecologist stalkers` |
| `alignment_conquer_mutant` | `non-cowardly mutants` |
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

Family with multiple specific causes (area, stash, needs, instincts): slide banner at the top (no umbrella tag), then for each specific cause: a self-contained sub-cause tag (`Cause:` line) and the cause's settings (enable toggle, any branch-only sliders), separated by line dividers.

Drive-grouped families (needs, instincts): threshold slider is per-drive, while enable toggles are per-answer (a drive may have multiple answers, e.g. slumber has slumber_field/slumber_lair/slumber_surge). Threshold sliders are rendered together at the end of the page after a `div_drives` divider, one per drive. Caller-facing label on each threshold tells the user which drive it affects.

Family with a single cause (alpha, alphakill, massacre, squadkill, basekill, wounded, harvest): just one tag, no sub-divisions.

The per-cause budget (`cause_max_<category>`) is exposed in the Framework MCM tab, not duplicated on family pages.
