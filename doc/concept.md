# Cause Pipeline — Concept

Working design captured 2026-05-02. Source for architecture.md. Both radiant and reactive shapes go here.

---

## Nomenclature (LOCKED)

These four words are LOCKED. No variation.

| Term | Meaning |
|---|---|
| **GENERATOR** | The file. Holds N specific causes. Picker. Emits one cause per call or none. **MUST NOT** be called a cause, umbrella, predicate, or anything else. |
| **RULES** | Yes/no checks that filter at one of the locations defined below. Toggle, alignment, personality, world-state predicate, threshold. Always ordered cheapest-first (toggle, alignment, world-state, personality roll). Fail → cascade. Locations differ by pipeline: radiant has one location (per cause); reactive has three (cause-level event, consequence-level event, per-responder). |
| **SCAN** | World lookup that produces a thing — a smart (destination) or a list of squads (responders). Replaces EVAL and FIND. **MUST NOT** be called EVAL or FIND or "find_smart" or "find_squads" at the design level. Two types: **destination SCAN** (locates a target smart) and **responder SCAN** (locates responder squads). Reactive consequences may use both; radiant causes use only destination SCAN. |
| **ACTION** | The consequence's effect: dispatch (`script_squad` / `script_actor_target`), state mutation, news record, on_arrive callback. The only thing reactive RESPOND and TRANSFORM consequences and radiant consequences all share. |

Pre-1.5.0 `cause:needs` confusion came from calling the file a cause. **Forbidden.**

---

## Radiant invariants (HARD RULES — no variation)

1. Generator **MUST** emit one specific cause per call, or none.
2. Actor is **always** the ticking squad. No actor scan in radiant.
3. Cause:consequence is **always 1:1**. **Never 1:N.**
4. SCAN **MUST** live in the generator. Never in the consequence.
5. Consequence holds **only** the ACTION (script_squad + on_arrive). No RULES, no SCAN.
6. Every cause **MUST** have its own MCM toggle. **No** file-level master toggle.
7. Generator **MUST** cascade internally (cause-to-cause) and externally (generator-to-generator at producer).
8. Total checks per radiant tick **MUST** be capped.
9. **No fallbacks inside a cause.** A cause has exactly one SCAN with one filter. If the SCAN fails, the cause returns and the cascade picks up the next candidate. **MUST NOT** implement try-filter-A-then-filter-B inside a single cause. Variant outcomes are separate causes.

---

## Radiant pipeline (MANDATORY FLOW)

The generator **MUST** execute these steps in this exact order:

1. **HULL** (bundle-only: needs, instincts) — score every cause, emit ranked overdue list. Hull does **not** pick. It produces the candidate list.
2. **INTERNAL CASCADE** — walk the candidate list top-down. For each candidate:
   - **RULES** (per cause): cheapest-first — toggle, alignment, world-state predicate, personality roll. Fail → next candidate.
   - **SCAN** (per cause, destination type): locate destination smart for the ticking squad. Empty → next candidate.
   - Both pass → publish this specific cause. **STOP.** Tick is done.
3. **EXTERNAL CASCADE** — candidates exhausted (all failed RULES or SCAN) → producer moves to next generator's candidate batch. The per-tick cap counts every check across this failure chain. Cap hit → tick ends with no publish.

No step may be skipped. No step may be reordered. No step may be merged with another.

Concept-scope checks (mutant for instincts, human for needs) are part of each cause's RULES. They run cheaply at the start of the per-cause RULES block. No separate FILTER step.

---

## Cap on radiant cascade

Max 5 checks per radiant tick across all generators combined. One check = one candidate running RULES + SCAN. Constant `RADIANT_MAX_CHECKS_PER_TICK = 5` in `ap_core_const.script`. Not in MCM.

## Per-CATEGORY rate limit

Producer-level, outside generators. Sliding window per cause type. Independent of the per-tick cap.

## Cause types (LOCKED)

Only two cause types exist. No others.

| Type | Trigger | Actor | Cardinality |
|---|---|---|---|
| **REACTIVE** | engine event (death, item use, ...) | N witnesses scanned per cause | 1:N |
| **RADIANT** | radiant tick on the ticking squad | the ticking squad | 1:1 |

Enum: `CAUSE_TYPE = { REACTIVE, RADIANT }` in `ap_core_const.script`.

## Consequence file shapes (LOCKED)

Only two shapes exist. No others.

| Shape | File pattern | When to use |
|---|---|---|
| **`_set`** | `ap_ext_consequence_<family>_set.script` | Hand-written N handlers in one file. Use when handlers have meaningful per-handler quirks the factory can't express. |
| **CONFIGS factory** | `ap_ext_consequence_<family>.script` | One CONFIGS table with N entries; closure builder generates N handlers from a single body. Default choice when handler bodies are uniform. |

**Pipeline preferences:**
- **Radiant: CONFIGS factory is the default.** Under the locked concept, every radiant consequence body collapses to: open trace → script_squad → check res → record → SUCCESS. Differences are pure data (cause, pre_release_gulag, on_arrive function, on_arrive_args). `_set` is vestigial for radiant; use only for genuine per-handler quirks.
- **Reactive: both valid; pick by body uniformity.** RESPOND consequences vary by SCAN filters and per-responder logic; TRANSFORM consequences vary by mutation. `_set` for hand-written quirks; CONFIGS factory only when bodies are uniform across entries.

Single-consequence files (1 handler) use `_set` shape with one entry.

## Result codes (LOCKED)

Reuse the existing `RESULT` enum in `ap_core_const.script`, with one rename: **`FAILED_EVAL` → `FAILED_SCAN`** codebase-wide. Aligns enum field with locked nomenclature (SCAN, not EVAL).

| Code | Meaning | Where it can be returned |
|---|---|---|
| `SUCCESS` | publish (cause) / consequence ran | cause + consequence |
| `FAILED_RULES` | RULES failed → cascade to next candidate | cause + consequence |
| `FAILED_SCAN` | SCAN failed → cascade to next candidate | cause + consequence |
| `FAILED_ACTION` | script_squad / on_arrive failed | consequence only |
| `DISABLED` | MCM toggle off, consumer pre-gate | consumer only |

Cause-level cascade reads only `SUCCESS` (stop) vs anything-else (next candidate). The `FAILED_*` distinction is for diagnostics, not flow control.

## Radiant consequence (MANDATORY SHAPE)

The consequence holds only the ACTION. Per-publish handler:

1. Open trace span.
2. `script_squad(squad, smart, { rush, on_arrive, on_arrive_args, pre_release_gulag })`. Squad and smart arrive resolved in the payload — no lookup, no SCAN.
3. `res.code ~= SUCCESS` → return `FAILED_ACTION`.
4. `record_event` (news).
5. Return `SUCCESS`.

**No RULES. No SCAN. No filter. No find. No alignment. No personality.** All of that lives in the cause.

## on_arrive contract (radiant)

Fires when the radiant-driven squad reaches the destination smart. Reactive consequences may also use on_arrive (e.g. chase-target re-targeting in alphakill_targeted), but the contract differs and is per-consequence.

- **DTO reset is unconditional.** The need / instinct is satisfied by arrival, regardless of online/offline state.
- **World mutation runs only when online.** Item consumption, trade, inventory operations require the live `game_object`. Offline → skipped (offline simulation already advances the abstract state via DTO reset).
- The "passive" branch (no section, no arrive_fn) is valid — DTO reset alone IS the satisfaction for some causes.

## SCAN-fail backoff (per-cause)

When the cause's SCAN fails on a candidate, the cause cascades to the next candidate. Optionally the cause may also apply a per-cause backoff (e.g. needs reset the DTO timestamp on SCAN-fail so the drive deprioritizes next tick on a level where no destination will ever match). This is per-cause behavior, not a pipeline rule.

## Why fails are designed-in

Per-cause RULES (especially personality) make outcomes more or less likely. Many attempts fail by design. Failure feeds the cascade. Variance comes from cascade ordering, not from clamping personality.

---

## Reactive invariants (HARD RULES — no variation)

1. Reactive cause **MUST** subscribe to either one xray engine callback OR one callback-family constant (see "Callback families" below). Cause publishes one specific cause string on threshold/condition met, or none.
2. Cause:consequence is **always 1:N**. Multiple consequences fire per cause publish.
3. RULES live in **three locations** (see "RULES locations" below). Each location filters at a distinct stage.
4. Reactive consequences come in **two shapes**: RESPOND (SCAN responders, per-responder loop) and TRANSFORM (act directly on the cause's payload entity, no SCAN). See "Consequence shapes" below.
5. RESPOND consequence's responder SCAN returns up to N squads (per-consequence configurable max_count, default 2).
6. RESPOND consequence cascades through SCAN responders, then control moves to next consequence.
7. Total consequences processed per cause publish **MUST** be capped.

## Callback families

Some logical events have multiple engine paths (e.g. an item pickup fires `actor_on_item_take` for the player and `npc_on_item_take` for an NPC). Treating these as one cause requires a callback-family constant.

Mirror the existing `RADIANT_CALLBACKS` shape: per-cause constant tuple in `ap_core_const.script`. Examples:
- `HARVEST_CALLBACKS = { ACTOR_ON_ITEM_TAKE, NPC_ON_ITEM_TAKE }`
- `WOUNDED_CALLBACKS = { X_NPC_MEDKIT_USE, ACTOR_ON_ITEM_USE }`

The cause's `on_game_start` registers the same predicate against every callback in the family. The predicate emits the same cause string regardless of which path triggered.

## RULES locations (LOCKED)

Reactive RULES split across three locations. Each filters at a distinct stage:

| Location | Filters | Runs | Examples |
|---|---|---|---|
| **Cause-level event RULES** | universal event criteria — does this event qualify for publishing AT ALL? | once per event | basekill: alignment_human victim + at_base + threshold |
| **Consequence-level event RULES** | per-consequence event criteria — does THIS consequence apply to this event? | once per consequence dispatch | basekill_flee: `alignment_flee` subset (`alignment_human` minus `alignment_principled`) + flee personality (INV_DISCIPLINE, INV_TERRITORY) on victim faction |
| **Per-responder RULES** | filter responders by individual identity — varying personality, species | once per responder during cascade | wounded_hunt: per-squad species check (`alignment_mutant_predator` + `aberrant`) |

A consequence may use any subset:
- RESPOND consequences with same-faction responders (basekill_flee, massacre_investigate) typically use cause + consequence-event RULES, no per-responder (responders share the event's faction profile).
- RESPOND consequences with mixed-identity responders (wounded_hunt, alphakill_targeted) typically use cause + per-responder RULES (per-responder species/personality filtering).
- TRANSFORM consequences typically use cause + consequence-event RULES (no responders to filter).

## Consequence shapes (LOCKED)

| Shape | What it does | SCAN | Examples |
|---|---|---|---|
| **RESPOND** | Locates responder squads, dispatches action per responder. | responder SCAN (`find_squads`) + optional destination SCAN (`find_smart`) | massacre_investigate, basekill_flee, wounded_hunt, alphakill_targeted, harvest_rob, etc. |
| **TRANSFORM** | Acts directly on the entity in the cause's payload. No SCAN, no responder pool. | none | alpha_promote (transforms the killer into an alpha) |

Both shapes are first-class reactive consequence shapes. Neither is a workaround for the other.

## Destination SCAN vs responder SCAN

Some RESPOND consequences need a destination smart in addition to responders (e.g. basekill_flee: find a faction base far from the kill site, then find responders to send there).

These run **two SCANs**:
- **Destination SCAN** (`find_smart` with filter) — locates the target smart.
- **Responder SCAN** (`find_squads` with filter) — locates responders to send.

Both SCANs are part of the consequence's SCAN phase. They run in whichever order the consequence requires (often destination first, since responder SCAN may use destination to exclude already-arrived squads).

## Reactive pipeline (MANDATORY FLOW)

**Cause side:**
1. Generator subscribes to xray callback (or callback-family).
2. On event: **cause-level event RULES** (event payload validation, threshold check, victim/actor alignment, world-state checks).
3. Conditions met → publish with payload. Else return.

**Consequence side (per consequence, until cap hit):**
1. **Consequence-level event RULES**: does THIS consequence apply to this event? Fail → next consequence (no SCAN, no work).
2. **SCAN** (for RESPOND shape):
   - 2a. Destination SCAN if applicable (`find_smart` with filter).
   - 2b. Responder SCAN (`find_squads` with filter, max_count=N).
3. For each responder, top-down (RESPOND only):
   - 3a. **Per-responder RULES**: alignment subset, personality, species. Fail → next responder, no action.
   - 3b. **ACTION**: `script_squad` / `script_actor_target` / `record_event`.
4. TRANSFORM shape skips 2 + 3, runs ACTION directly on cause-target from payload.
5. After consequence completes, cascade to next consequence.

## Cap on reactive cascade

Max consequences processed per reactive cause publish. One "check" here = one consequence run (its full RULES + SCAN + per-responder loop OR transform), independent of how many responders it had.

**Proposed value: constant `REACTIVE_MAX_CONSEQUENCES_PER_CAUSE = 5`** in `ap_core_const.script`. Symmetric with the radiant cap, distinct variable. Internal pipeline tuning, not MCM.

Distinct from the radiant cap:
- Radiant cap counts candidate causes checked across all generators in one tick. Stops on first publish OR cap.
- Reactive cap counts consequences run per cause publish. No stop-on-success — runs every consequence up to cap.

## Reactive consequence shapes (MANDATORY)

**RESPOND shape** — per-publish handler:

1. Open trace span.
2. **Consequence-level event RULES**. Fail → return FAILED_RULES.
3. **Destination SCAN** if applicable (`find_smart`). Empty → return FAILED_SCAN.
4. **Responder SCAN** (`find_squads`, max_count=N). Empty → return FAILED_SCAN.
5. For each responder:
   - 5a. **Per-responder RULES**. Fail → continue.
   - 5b. **ACTION**. Track success.
6. No responder passed RULES → return FAILED_RULES.
7. At least one acted → return SUCCESS.

**TRANSFORM shape** — per-publish handler:

1. Open trace span.
2. **Consequence-level event RULES**. Fail → return FAILED_RULES.
3. Validate cause-target entity (from payload).
4. **ACTION** on cause-target (state mutation, registration, hit-power update, news record).
5. Return SUCCESS / FAILED_ACTION.

Reactive consequences may legitimately differ in body shape (different SCAN filters, different action types, different per-responder logic). `_set` is the natural choice for hand-written quirks; CONFIGS factory works only when bodies are uniform.
