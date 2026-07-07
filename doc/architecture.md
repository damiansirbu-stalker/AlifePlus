# AlifePlus Architecture

AlifePlus is a reactive framework for STALKER Anomaly. It hooks X-Ray engine callbacks, classifies them through cause generators, and dispatches consequences. Two pipelines:

- Event Pipeline (ap_core_producer): gate chain -> cause generator -> xbus publish.
- Dispatch Pipeline (ap_core_consumer): xbus subscribe -> consequence handler -> script_squad.

The framework produces its own radiant heartbeat (ap_core_callbacks.script declares and fires ap_squad_on_change at the MCM-configured rate and map mix) and owns protection (ap_core_broker), rate limiting (ap_core_limiter), squad lifecycle (ap_core_broker), tracing (ap_core_debug), PDA map markers (ap_core_map), and statistics overlay (ap_core_hud). Domain logic registers a cause generator with the producer and a consequence handler with the consumer.

Two layers: core (ap_core_*) and ext (ap_ext_*). Core never imports ext. All domain logic reaches the framework through registered function references.

Built on xlibs. _ap_deps asserts xlibs presence and version on load. See conventions.md for naming, result codes, MCM, logging.

Part of a three-mod alife family: **AlifePlus** extends A-Life with new behaviors (this mod), **AlifeBalance** tunes existing rates and counts, **AlifeGuard** keeps alife state clean.

![System Overview](img/system-overview.png)

---

## Invariants

- **No steady-state per-frame work.** Ongoing work runs on a throttled tick (a fixed interval) or on a discrete engine event (hit, shot, spawn, option change); it never runs continuously every frame. A per-frame engine callback (`npc_on_update`) is used only as a carrier that throttles before doing anything, and we never place our code on a path the engine runs every frame (a visibility or fire functor). Frame-spreading a bounded one-off batch (xslice, 1 item per frame) to avoid a single-frame spike is the one allowed use of the frame; it completes and stops. Full rule and rationale: `doc/standards/code-standards.md` "No Per-Frame Work".

---

## Decision model

AlifePlus extends A-Life on the input side and replaces it on the output side. It consumes the same world the vanilla simulation runs on (engine callbacks, smart props, squad rosters) but writes a different kind of decision into it.

### The control field

Every squad runs `sim_squad_scripted:update` every tick, and one branch decides everything (`sim_squad_scripted.script:233-238`):

```lua
local script_target_id = self:get_script_target()
if (script_target_id) then self:specific_update(script_target_id)
else self:generic_update() end
```

`get_script_target()` returns `self.scripted_target` first when it is set (`sim_squad_scripted.script:104-106`). That single field is AlifePlus's entire write surface into targeting. AP's radiant input is not the engine heartbeat: the `ap_core_callbacks` sweep (installed at on_game_start) visits every squad on a staggered timer, diffs a 4-field snapshot, and produces `ap_squad_on_change` fires at the configured rate and map mix, real transitions first, round-robin turns for quiet squads (see Event Pipeline). The pipeline runs synchronously inside that timer tick: sweep fires `ap_squad_on_change` -> gates -> cause generator -> `ap_core_broker.script_squad` -> `xsquad.acquire_squad` sets `scripted_target`. The squad consumes the write at its NEXT `sim_squad_scripted:update`: the fork at `:233` reads `scripted_target` first, one frame later for online squads, up to one scheduler visit (~30s) later for offline squads. This is the same timing contract every reactive dispatch has always had (death and item callbacks also run outside the target squad's update), so radiant and reactive now share one synchrony model. AP authors only the write-back; the engine's own update loop acts on it.

### Vanilla: a stateless weighted lottery

`scripted_target` nil -> `generic_update` -> `SIMBOARD:get_squad_target` (`sim_board.script:379-423`): score every available target with `evaluate_prior`, keep the top 5, then roll - 50% the single highest, 50% a random one of the top 5 (`sim_board.script:418`). The score (`simulation_objects.script:252-343`) is `(base + faction props + category weights) * (1 + 1/graph_distance)` (`simulation_objects.script:177-183`). Only the distance term varies per squad; `target.props[squad.player_id]` is faction-keyed and `default_squad_behaviour` / `default_monster_behaviour` are species-keyed. Two squads of one faction at the same position score the entire board identically and differ only by the coin flip. A vanilla squad carries no internal state - it is a position plus a faction label, pulled toward smarts by an inverse-distance attraction field shared across its whole faction.

### AlifePlus: a per-squad stateful command

`scripted_target` set -> `specific_update` (`sim_squad_scripted.script:276-331`): the squad walks straight to the assigned smart - no `evaluate_prior`, no top-5, no roll. The destination is computed from that squad's own accumulated state: its hunger and instinct timers, opportunity windows, personality roll, and post-scan tactics, all held in DTOs keyed by `squad_id` (`ap_ext_tracker.script:167-205`). AP suppresses the lottery for that squad until the broker releases `scripted_target`, after which the squad falls back into `generic_update`.

The distinction is bias versus command. Vanilla writes a bias - faction-wide props that tilt a draw each squad makes for itself. AlifePlus writes a command - a concrete target id that overrides the draw for one squad.

### Granularity: squad, not NPC

Both systems decide at the squad level. AlifePlus adds no per-NPC targeting; the DTOs and `scripted_target` are squad-scoped. NPC-level individuality already exists in vanilla and lives in the gulag: on arrival, `select_npc_job` (`smart_terrain.script:626-798`) assigns each member a distinct job (camper, sleeper, animpoint) by priority and exclusivity, then `setup_logic` activates its scheme and the GOAP planner. Vanilla migration, AP migration, and the gulag all converge here. AP commands where a squad goes; the gulag still owns what each NPC does there, and AP never touches it (see Engine integration -> Write surface).

---

## Vocabulary

- Cause: a labeled event the framework publishes. Specific name (cause:massacre, cause:hunger_campfire). Umbrella names (NEEDS, REACTIONS) are categories, never published.
- Consequence: a handler subscribed to a cause; runs the action and side effects.
- Cause generator: function that picks and emits one cause per call, or none. One generator per family. Files named ap_ext_cause_<family>.script for single-cause, ap_ext_causes_<family>.script for multi-cause.
- Cause type: RADIANT (squad-tick driven, 1:1 with consequence) or REACTIVE (engine-event driven, 1:N with consequences). Defined in ap_core_const.CAUSE_TYPE.
- Cause category: REACTIONS, NEEDS, INSTINCTS, OPPORTUNITIES. Behavioral axis parallel to cause type. Drives per-category rate-limit grouping and MCM organization. Defined in ap_core_const.CAUSE_CATEGORY.
- RULES: business checks that need no world scan. Toggle, alignment, species, personality, payload field, threshold, tactics. Cheap. Always cheapest-first.
- SCAN: world lookup. find_smart, find_squads, find_stashes. Two flavors: destination SCAN (locates a target smart) and responder SCAN (locates responder squads).
- ACTION: the consequence's effect. script_squad / script_actor_target, state mutation, news add, on_arrive callback.
- Hull(<drive>): Hull's drive reduction theory (1943). Score = weight * (elapsed / threshold)^2 (ap_ext_causes_needs.script:96-98). Squared exponent makes overdue drives compete strongly. Used by NEEDS (9 stalker drives) and INSTINCTS (5 mutant drives).
- MVT(<cause>): Charnov's marginal value theorem (1976). Binary patch-recovery gate: elapsed > threshold. Used by stash and area causes (6 OPPORTUNITY fields).
- personality(<TRAITS>): probability check on a faction or species. avg(traits) clamped to [PERSONALITY_FLOOR, PERSONALITY_CEILING], rolled per dispatch. Inverted traits resolve as 1 - base before averaging.
- tactics(<reason>): post-SCAN soft check on radiant decisions. Score starts at 1.0; each tactical concern subtracts a weight. math_random() > score returns FAILED_RULES with reason LOW_TACTICS. Conditions: last-of-faction at source (-0.20), smart already targeted by another scripted squad (-0.20). Radiant only; reactive bypasses (event responses are not voluntary picks). Implemented in `ap_ext_util.is_tactically_eligible`.
- Cross-DTO read: any cause generator may read any DTO; only the owning generator writes.
- Producer (ap_core_producer): runs the gate chain, evaluates cause generators, publishes to xbus.
- Consumer (ap_core_consumer): subscribes to xbus events, dispatches to consequence handlers.
- xbus: pub/sub event bus from xlibs. Causes publish, consequences subscribe.

---

## Invariants

Project-wide constraints that hold across all pipelines and modules.

- Core never imports ext. All ext modules reach core through registered function references at on_game_start.
- Cross-DTO read, single-writer write. Any cause generator may read any DTO; only the owning generator writes.
- Performance budget. Every measured flow (each bracketed `ap_core_debug.observe` label in the log: `[CONSEQUENCE_PHASE.FIND_TARGETS]`, `[CONSEQUENCE_PHASE.FIND_DESTINATION]`, `[STASH]`, ...) targets 0.1ms average per call and has a hard 4ms ceiling per call. No exceptions, including cold start, save load, and level transition. Any preload (index build, cache warm, registry walk) that would breach 4ms in a single tick must be frame-spread (xslice or equivalent), with the search path falling back to the un-indexed walk until the build completes. A flow that averages above 0.1ms, or that ever exceeds 4ms in a single call, is a regression and requires a perf task. See `doc/standards/code-standards.md` "Performance budget".
- Item transfer only. AlifePlus never invents new section names. Every materialized item is a section that already exists in vanilla configs. AP policy LTX files (`ap_trade_policy.ltx`, `ap_stash_policy.ltx`, `ap_loot_policy.ltx`, `ap_market_policy.ltx`) declare category bands; the actual sections come from `xinventory.get_category_sections(category)`, which reads vanilla Parse_ITM buckets (`_ITM["outfit"]`, `_ITM["ammo"]`, etc.) and hand-curated sets of real vanilla sections (`_medkit_set`, `_grenade_set`, etc.). Some sections come from `character_desc [supplies]`, `trade_*.ltx [supplies_N]`, or `treasure_manager` tier configs. None are invented by AP.
- Money flows match vanilla. NPCs gain money when they sell items above category max (`floor(cost * 0.5)`) and spend money when they buy items below category min (`floor(cost * 1.0)`). Cost reads `xinventory.get_cost(sec)` which wraps `ini_sys:r_float_ex(sec, "cost")`. Money moves only inside `ap_ext_trade.trade` via `xcreature.transfer_money` and `xcreature.give_money`, both defensive against the engine `TransferMoney` u32 underflow (see todo-demonized-exes.md n015 PR). No money moves through stash loot, stash fill, the faction market, or any other AP path. The market's premium is not an AP money move: it overrides the engine's displayed price for tagged trader stock (`on_get_item_cost`), and the player pays it through the normal engine buy.

---

## File layout

Two layers. Core is the framework. Ext is the domain. Core never imports ext; all ext modules register with core via function references at on_game_start.

### Core (15 files)

| File | Role |
|------|------|
| _ap_deps | Dependency gate: assert xlibs installed and version-compatible |
| ap_core_const | Enums and timing constants: CALLBACK, CAUSE_TYPE, CAUSE_CATEGORY, RESULT, REASON, TRACE, RANGE_*. |
| ap_core_mcm | MCM defaults, cfg snapshot, UI builder, on_option_change |
| ap_core_debug | Logger, observe() tracing, bracket helper, result builders. Zero overhead below DEBUG |
| ap_core_cache | Per-level indexes over SIMBOARD.smarts_by_names (sync at actor_on_first_update), treasure_manager.caches (xslice step=3, ~2s warmup), and SIMBOARD.squads (sync at actor_on_first_update). Consumers fetch via smarts_on_level(level_id) / stashes_on_level(level_id) / squads_on_level(level_id) and pass as opts.source to xsmart.find_smart / xstash.find_stashes / xsquad.find_squads. Smarts and stashes never invalidate (LTX-baked positions). Squads update incrementally via squad_on_after_level_change (move between buckets) and server_entity_on_register / on_unregister (spawn / despawn); stale entries are a no-op cost because xsquad._squad_base_valid level gate is first. Also builds a standalone-role index keyed by smart_id (cross-level). Populated by xslice over the full alife pool via sim:iterate_objects with per-entity Stalker + Trader clsid class-gate. For each curated NPC (section in xdata.npc_roles with trader / medic / mechanic role) resolves the owning smart via smart.npc_info (engine roster) with find_smart-by-position fallback, then inserts the NPC into every applicable role bucket per xdata curation (multi-role NPCs land in trader AND mechanic when xdata declares both). Consumers: standalone_role_at_smart(smart_id, role) returns id-set, first_online_role_npc_at_smart(smart_id, role) returns first live game_object. Full rebuild per level transition (no incremental upkeep on this bucket; Lua VM reinit + actor_on_first_update fires fresh build). The earlier 1.7.1 approach using alife():iterate_level_objects_of_clsid walked graph().level().objects() (engine's switched-in registry, not the per-level alife pool) and silently dropped most offline curated NPCs |
| ap_core_util | xbus pub/sub wrappers, find_smart / find_squads with protection filters |
| ap_core_limiter | Rate-limit primitives. Pipeline family (real-sec, ephemeral): per-key cause counter, per-consequence token bucket, global radiant TTL counter. Balance family (game-sec, persisted): offmap dispatch counter |
| ap_core_callbacks | Synthetic callbacks AP declares and fires: ap_squad_on_change (a staggered SIMBOARD.squads sweep producing a rate/mix-controlled squad heartbeat, reads cfg.alife_rate/alife_ratio live) and ap_npc_medkit_use (xevent.hook on xr_eat_medkit.consume_medkit). Both declared at module load, installed in on_game_start. Tracing complete but one-boolean-gated on ap_core_debug.enabled() |
| ap_core_producer | Event Pipeline: radiant gate chain on ap_squad_on_change (budget peek, protection), reactive gate chain on engine callbacks, cause generator cascade, xbus publish |
| ap_core_consumer | Dispatch Pipeline: xbus subscribe, consequence iteration, result codes, rate gating |
| ap_core_broker | Squad lifecycle: protection, ownership registry, scripting, 20s scan, arrival, release, transit balance |
| ap_core_chase | Squad-target chase: two guarded sim_squad_scripted monkeypatches (get_current_task / get_script_target) that home an AP-scripted chaser on another squad by id, keyed on the broker registry; delegate to the captured original for non-chase squads (chain-safe with warfare) |
| ap_core_anomaly_fixes | Vanilla Anomaly bugfixes. FIX 1: SIMBOARD roster desync -- wraps sim_squad_scripted specific_update / generic_update to call xsmart.reconcile_squad_roster after each (the roster-blind vanilla assign_smart leaves SIMBOARD.smarts[].squads / population stale). WRAP not replace; idempotent, so inert where a base already fixes it. `_patched` guard + class nil-guard |
| ap_core_record | Activity record: FIFO of dispatched consequences with denormalized display facts; HUD / news / external integrators read from it |
| ap_core_map | PDA map markers rendered from activity record entries; 10s apply timer, 20s validate pass, 60s linger before unmark |
| ap_core_hud | Pipeline statistics overlay (UIStatsHUD): counters, gate breakdown, periodic dump |
| ap_core_compat | Save data cleanup for version upgrades, default ownership proxy (warfare) |

### Ext: cause and consequence files

Single-cause files (7), one cause per family: ap_ext_cause_alpha, ap_ext_cause_alphakill, ap_ext_cause_basekill, ap_ext_cause_harvest, ap_ext_cause_massacre, ap_ext_cause_squadkill, ap_ext_cause_wounded.

Multi-cause files (4): ap_ext_causes_area, ap_ext_causes_instincts, ap_ext_causes_needs, ap_ext_causes_stash.

Consequence files (11, always plural): ap_ext_consequences_alpha, ap_ext_consequences_alphakill, ap_ext_consequences_area, ap_ext_consequences_basekill, ap_ext_consequences_harvest, ap_ext_consequences_instincts, ap_ext_consequences_massacre, ap_ext_consequences_needs, ap_ext_consequences_squadkill, ap_ext_consequences_stash, ap_ext_consequences_wounded.

### Ext: named modules (12 files)

| File | Role |
|------|------|
| ap_ext_const | Domain const tables: CAUSE_CATEGORY, alignment_*, PERSONALITY traits, mutant species displays, range tiers |
| ap_ext_util | Domain gates: alignment / species / personality / tactics checks, FIFO-cached species resolution |
| ap_ext_common | Shared chase dispatch: move_actor_chasers (player target) / move_squad_chasers (squad target) |
| ap_ext_tracker | Domain state: kill counts, alphas, alpha-dead grace, stalker NEEDS DTO, mutant INSTINCTS DTO, squad OPPORTUNITY DTO |
| ap_ext_smart_mutator | Runtime smart terrain mutations: territory conquest (shared spawn) and mutant infestation (exclusive spawn) |
| ap_ext_trade | NPC supply-trader orchestration: loads `configs/alifeplus/ap_trade_policy.ltx` per-rank blocks (`[ap_trade_policy_rookie]` / `[ap_trade_policy_veteran]` split by `RANK_VETERAN`) for per-category min / max; per-event net-profit cap is the MCM `trade_profit_max` override (economy > trade), applied to all ranks |
| ap_ext_loot_claim | Three-way corpse loot ownership over one xttltable ledger (victim -> {killer, squad, death game_sec}; actor kills via npc_on_hit `from_death_callback`, NPC kills squad-owned via `get_object_squad` with lone-killer fallback). Half A (reserve own) and Flow C (npc-vs-npc) share a generic exclusion arbiter `_npc_loot_excluded` xevent-hooked at `xr_corpse_detection.near_actor`; Flow C's scanning-NPC identity comes from a `find_valid_target` wrapper (xevent on the global `evaluator_corpse` class), since near_actor receives only the corpse. Half B vetoes the actor opening a claimed kill at `on_before_key_press` (level.get_target_obj), with `ActorMenu_on_item_before_move` and Dot Marks backstops. Each flow carries its own MCM enable / radius / TTL; the death timestamp is aged per-flow at read, ledger retention capped at the max TTL. Squad claim holds while a living member sits within radius (`xsquad.iter_member_ids`); squad id validated against recycling via `xsquad.is_squad`. Warn and yield tips draw from three per-flow pools; two variants per pool carry a `#name#` slot (the news token convention) filled by the looter or owning-NPC name, variant 001 kept nameless as the fallback |
| ap_ext_market | Faction market: observes `ap_ext_trade.trade` output into a per-faction bounded dedup FIFO of recently-sold sections, and at a curated faction trader's restock (one of two detection paths) injects a rank-gated (`xinventory.get_rank_chance`), role-fit, distribution-claimed, capped echo at an MCM-set condition, priced at an MCM premium via the engine `on_get_item_cost` hook, released on a window timer. Read-side of the loot/trade/market loop; no AP money path |
| ap_ext_loot_select | Corpse loot policy (sibling to loot_claim: claim decides who may loot, select decides what is kept). Subscribes to `npc_on_get_all_from_corpse` (per-item, fired before `corpse:transfer_item`), records the looted item ids, and runs one trim deferred to the next frame via `CreateTimeEvent` (idempotent per looter id). The trim reuses the xinventory kernel (`load_policy [ap_loot_policy]`, `classify`, `policy_key`, `get_cost`, `release_item`): per capped key it releases the cheapest looted members first until the whole-inventory count is at/under max, keeping the highest-value loot and leaving any standing excess to the AlifeBalance cull. Untouchable / equipped inherited from `get_category`. MCM `loot_select_enabled`; online only; skips unscriptable looters. No money path, no save state |
| ap_ext_fx | Multi-source horror aggregator: composes xpp psy_antenna tint, xsound tinnitus, and level snd_volume ducking via max-across-sources factor; consumers call add_source / remove_source / clear; acquires xpp + xsound slots on first source, releases on last clear |
| ap_ext_news | News composer: per-consequence templates, slot substitution, speaker selection, dynamic_news_manager dispatch |
| ap_ext_test | In-game debug commands |

---

## Pipeline

### Execution model

X-Ray runs Lua single-threaded. There is no concurrency within one engine tick. When a callback fires (the sweep's ap_squad_on_change, or an engine reactive callback), the entire AP pipeline executes synchronously in one Lua call stack before control returns to the engine. Producer evaluates gates, cause generator runs, xbus publishes, consumer iterates consequences, each handler runs and may script a squad. All inside a single function call.

```
sweep tick (1s timer, 20 squads, cursor)
  -> diff snapshot -> SendScriptCallback("ap_squad_on_change", squad, changed, prev, curr)
    -> producer._on_radiant (gate chain: BUDGET peek, is_protected, RATIO)
      -> EVAL cascade: shuffle generators, try each, stop on first publish
        -> cause generator returns nil -> try next
        -> cause generator returns { cause = CAUSE.X, ...payload } -> publish + break
          -> xbus.publish (synchronous, inline)
            -> consumer._process (shuffle consequences, iterate)
              -> consequence handler returns { code = RESULT.X }
                -> script_squad sets scripted_target, registers arrival
            <- returns to producer
          <- cascade breaks on first publish
```

xbus.publish calls each subscriber inline and returns when all have finished.

### Event Pipeline (ap_core_producer)

Producer receives engine callbacks and filters them through a gate chain before evaluating cause generators. Two variants because radiant and reactive events differ structurally.

Radiant events are ambient observations delivered by `ap_squad_on_change`, produced at a configured rate and map mix. The squad is both subject and responder. Every fire is meaningful: either an observed transition or the squad's round-robin turn.

Reactive events are world-state changes on engine callbacks (death, healing, pickup). The triggering entity may not be the entity AP acts on. Low frequency, high significance per event.

#### The ap_squad_on_change sweep (ap_core_callbacks.script)

AP does not subscribe to the engine's `squad_on_update` heartbeat (6,624 calls/min across ~776 squads, of which the old gate chain discarded 99.94%; the engine still fires it for other mods). The replacement is a custom synthetic callback AP declares and fires itself: `ap_core_callbacks.script` declares `ap_squad_on_change` into the vanilla `_g.script` callback registry (`AddScriptCallback` at module load, so the name exists before any subscriber binds) and installs the sweep in its own on_game_start. Rate and mix are read live from `cfg.alife_rate` / `cfg.alife_ratio` each tick, so MCM changes apply on the next tick with no push path.

The sweep separates detection from production:

Detection: a 1s time event walks 20 squads per tick with a circular cursor over an id snapshot from `xsquad.collect_squad_ids` (SIMBOARD.squads is engine-script truth, registered at `sim_squad_scripted.script:1010`, removed at `:1019`). Per-tick cost is O(batch), constant, independent of total squad count; more squads only lengthen the full pass (~776 -> ~40s). This mirrors the engine's own A-Life scheduler shape (20 objects per tick round-robin, `alife_schedule_registry.h:25-51`), one level up. Each visit resolves the id (`xobject.se`; ids, never held refs), tags the squad's lane (actor's current level vs background, `xlevel` cached lookups) into the next fire-order list, and diffs a 4-field snapshot; a changed squad is queued, not fired:

| Field | Signal |
|-------|--------|
| m_game_vertex_id | movement (pre-quantized position; the engine advances it as the squad walks) |
| online | online/offline transition |
| smart_id | assignment change |
| current_action | moving vs staying |

Production: a credit accumulator gains `cfg.alife_rate / 60` per tick; each whole credit emits exactly one fire, so the total rate is enforced by counting, not by timers. Per credit: the lane is picked by `cfg.alife_ratio` (the manifesto's Bresenham, current level vs background, applied at production instead of rejection; the same slider the reactive ratio gate uses); the lane's oldest queued change fires first; with no pending change, the lane's next squad fires round-robin — the implicit keepalive that guarantees every squad its turn, longest-waiting first by list order. Fire payload is `(squad, changed, prev, curr)`; `changed` names the differing fields, nil = round-robin fire with no transition. The fire re-diffs live fields, so a queued change that reverted goes out truthfully. One clamp: a per-squad refractory (60s, os.clock) protects small lanes from rapid re-fires; a blocked fire defers (snapshot not advanced, re-detected next pass). First sight of a squad baselines silently; spawn is reactive territory. Snapshot state is ephemeral (never saved); eviction via server_entity_on_unregister plus the resolve-time nil checks.

The sweep is detection-only: no protection, no budgets, no knowledge that a pipeline exists. It fires for protected and warfare-owned squads too, so the stream the producer sees is uncensored; `ap_squad_on_change` goes through the same vanilla callback registry as any engine callback, so any co-installed mod can subscribe to it with plain `RegisterScriptCallback` (the event exists only when AlifePlus is installed, since AlifePlus declares and fires it). Payload tables (`changed`, `prev`, `curr`) are reused between fires: read-only during the callback, never retain references. Instrumentation is complete but costs one boolean when DEBUG is off: the sweep gates every stat accumulator, `xprofiler.new_if` tick timing, and log line on `ap_core_debug.enabled()`; at DEBUG it emits one `[CALLBACKS.SQUAD]` line per fire (squad, lane, kind, field values) plus a per-pass summary (`visits, detected, fires cur/bg, change/keep, dropped, tick avg/max ms`) into alifeplus.log.

#### Radiant gate chain (ap_core_producer._on_radiant)

Two gates on each `ap_squad_on_change` fire; rate and level mix are enforced at production inside the sweep, so no ratio gate remains on the radiant path (the manifesto's A-Life Ratio lives at the intake now):

1. BUDGET peek. Non-consuming read of the global radiant dispatch counter (`ap_core_limiter.check_global_consequence_rate_limit`). Radiant is 1:1 publish-to-dispatch and the consumer refuses past `global_consequence_max_events` per window, so when the window is full every evaluation is provably wasted work and would violate the cause publish contract (universal rule 14). Window full -> return before any work. Self-tunes with the MCM cap; the counter only advances on dispatch SUCCESS, so failed consequences do not eat the window.
2. is_protected. Full eligibility check via ap_core_broker.is_protected. Checks ownership, scripted, permanent, active_role, task_target. Short-circuits early.
3. EVAL (cascade). Shuffles registered cause generators in-place inside a reusable buffer, walks them in random order. Each generator self-gates the per-CAUSE_CATEGORY rate (check_cause_rate_limit at the top of the predicate); the producer does not rate-check. First generator to publish stops the cascade. Each generator's `_on_smart` carries its own internal SCAN budget RADIANT_MAX_SCANS_PER_GENERATOR = 1 (`ap_core_const.script:147`); RULES rejections are free and the generator can cascade through all of its causes, but only the first cause that passes RULES and enters its world SCAN consumes the slot — any later cause that reaches SCAN in the same `_on_smart` call hits BUDGET_EXHAUSTED. Budget is a local Lua counter, resets every `_on_smart` call.

Shuffling ensures fair distribution regardless of how many generators apply to a given squad's alignment. Per-squad fairness comes from the sweep's round-robin production: every squad gets its turn instead of winning a global pacer lottery, so starvation is structurally impossible.

#### Reactive gate chain (ap_core_producer._on_reactive, line 259)

Reactive callbacks: squad_on_npc_death, ap_npc_medkit_use (synthetic, declared and hooked in ap_core_callbacks), actor_on_item_take, npc_on_item_take, actor_on_item_use. Three gates:

1. PACER. Token bucket with per-callback-type keys, 1 token/sec per type. Independent from the radiant pacer.
2. RATIO. Same Bresenham as radiant with its own counter pair. Some reactive callbacks (actor_on_item_use, actor_on_item_take) are always on-map (require a game_object which only exists online).
3. EVAL (per-handler walk). Iterates all reactive cause generators for this callback in registration order. Each generator self-gates its rate; the producer does not rate-check. All evaluate independently. A single death event can trigger MASSACRE, SQUADKILL, ALPHA generators in sequence. Reactive does NOT cascade-and-stop - every generator gets its chance to publish.

#### Protection layers

Protection is applied at four layers, same guard set against different entities depending on context:

| Layer | Where | Subject |
|-------|-------|---------|
| Producer | radiant gate 2 (is_protected) | triggering squad |
| Cause | reactive cause generators | the entity AP would act on (killer, patient, taker) |
| Consequence | ap_core_util.find_squads | every candidate responder squad |
| Squad | ap_core_broker.script_squad | no inline check; protection is upstream |

Reactive causes skip the producer protection gate because the callback entity (e.g. dead victim in squad_on_npc_death) is not the entity AP would script. They check protection on the relevant entity inside the generator: alpha and alphakill guard the killer, wounded guards the patient, harvest guards the taker. Causes whose trigger is a dead victim (massacre, squadkill, basekill) need no in-cause guard - consequences find responders through find_squads which applies all exclusions.

script_squad does not check protection. It assumes upstream layers verified the squad. Direct callers outside the pipeline must check is_protected themselves.

#### Cause generator contract

Takes the callback args, returns either a payload tagged with the picked specific cause or nil. Generators self-gate the per-category rate via ap_core_limiter.check_cause_rate_limit and increment_cause_counter; the producer does not gate cause budgets. Producer wraps every generator call in observe(trace, entry.name, entry.handler, ...) so rejection branches (no result.cause) still log. Single-cause reactives register name = CAUSE.X; multi-cause radiants register name = family string (stash, area, needs, instincts) and additionally wrap each per-cause attempt in observe(trace, cause_const, ...) so picked causes nest under the family.

```lua
local CATEGORY = CAUSE_CATEGORY.OPPORTUNITIES
local function _predicate(trace, squad)
    if not cfg.cause_X_enabled then return { code = RESULT.FAILED_RULES } end
    if not ap_core_limiter.check_cause_rate_limit(CATEGORY, cfg.cause_max_opportunities) then
        return { code = RESULT.FAILED_RULES, reason = REASON.BUDGET_EXHAUSTED }
    end
    -- RULES + SCAN, then on success:
    ap_core_limiter.increment_cause_counter(CATEGORY)
    return { cause = CAUSE.X, ...payload }
end
```

Generators are pure: no rate-limit awareness from outside, no counter manipulation, no side effects beyond the published payload.

#### Instrumentation

Both pipelines measure gate span and eval span. Gate span: time from first eligibility check to admission (radiant: BUDGET peek + is_protected + RATIO; reactive: PACER + RATIO). Eval span: cause generator evaluation, xbus dispatch, all consequence handlers. Spans accumulated per PACER_LOG_INTERVAL (60s) and reported as avg/max in the periodic [PIPELINE] dump. All timing uses os.clock() and is gated by debug enablement; zero overhead when log level is above DEBUG.

### Dispatch Pipeline (ap_core_consumer)

After a cause publishes to xbus, the consumer receives the event and iterates registered consequence handlers. Loop in ap_core_consumer._process; per-handler steps in _dispatch_entry.

#### Per-handler dispatch

Per consequence:

1. condition pre-gate. Function registered at consumer.register, typically the MCM enabled flag. False -> SKIP this consequence.
2. Per-consequence rate limit (ap_core_limiter.check_consequence_rate_limit). Exhausted -> SKIP.
3. Global radiant rate limit (radiant only, ap_core_limiter.check_global_consequence_rate_limit). Exhausted -> STOP the entire loop.
4. Handler runs. Returns { code = RESULT.X, reason = "..." }. FAILED_RULES (business rules rejected: alignment, personality, species, validation), FAILED_SCAN (rules passed but world query empty), FAILED_ACTION (rules and scan passed but the action failed: script_squad could not route, registration call returned false), SUCCESS.
5. On SUCCESS: increment per-consequence counter; for radiant only, increment global radiant counter; set event_data._fired = true. For radiant, also stop the loop (rule 5 below).

DISABLED is the rules-layer skip for a consequence whose MCM toggle is off. Semantically equivalent to a FAILED_RULES with reason="disabled"; emitted by the consumer pre-gate (`ap_core_consumer.script:87`) so the handler is not called. The four phase codes are still what the handler returns when it runs.

#### Dispatch rules

1. Radiant cascade (producer-side). Producer shuffles registered radiant generators and cascades through them in random order. First generator to publish stops the cascade. Remaining generators do not evaluate.
2. Cause publish contract. A cause generator must not publish if no consequence can act. Cascade stops on publish - if every consequence rejects, the trigger is wasted. Causes serving a subset of alignments must filter at the cause level (alignment_human for stash/needs, alignment_mutant for instincts) rather than relying on consequences to reject.
3. Per-need / per-instinct routing. Needs and instincts each register one generator with the producer but publish specific xbus events per need/instinct (cause:hunger_campfire, cause:heal_shelter, cause:scatter, etc.). Each consequence subscribes to its specific event. Only consequences that handle the winning need/instinct receive the event. No mismatch iteration.
4. Consequence shuffle. The consumer shuffles registered consequences before iterating so order is not fixed. Reactive causes have multiple consequences (e.g. massacre_investigate and massacre_scavenge); shuffling matters. Radiant causes have one consequence each (1:1 shared noun), so the shuffle is a no-op.
5. Radiant: at most one consequence runs per cause. 1:1 shared noun. The consumer runs it or skips it; no loop, no alternatives.
6. Reactive: all run independently. All consequences for the cause run regardless of results. Multiple consequences can fire from a single event.
7. Loop stop (reactive only). The reactive loop stops when REACTIVE_MAX_CONSEQUENCES_PER_CAUSE = 2 consequences have actually run. Other results (FAILED_RULES, FAILED_SCAN, FAILED_ACTION, DISABLED, rate-limited) skip and continue.

### Rate limiting

ap_core_limiter holds two limiter families. Pipeline limiter throttles event flow so the engine doesn't churn (real-sec clock, ephemeral). Balance limiter caps in-world impact so the world doesn't drift (game-sec clock, persisted across save/load). The other layers live in producer or in cause generators.

| Layer | Mechanism | Scope | Default | Lives in | Config |
|-------|-----------|-------|---------|----------|--------|
| Per-key cause counter | TTL counter, sliding window | per CAUSE_CATEGORY | 20 / 60s | ap_core_limiter (check_cause_rate_limit, increment_cause_counter) | MCM cause_max_<category> |
| Per-consequence token bucket | peek / acquire | per consequence name | 2 / 60s | ap_core_limiter (check_consequence_rate_limit, increment_consequence_counter) | MCM consequence_max_events |
| Global radiant TTL counter | TTL counter | radiant only | 5 / 60s | ap_core_limiter (check_global_consequence_rate_limit) | MCM global_consequence_max_events |
| Offmap balance counter | TTL counter, sliding window (game-sec) | per source level_id | 2 / 48 game-hours | ap_core_limiter (check_offmap_rate_limit, increment_offmap_counter) | MCM cause_max_offmap |
| Per-causer reaction counter | TTL counter, sliding window (game-sec) | per causer entity id (shared across dispatching REACTIONS; alpha promotion excluded) | 1 / 1 game-hour | ap_core_limiter (check_causer_rate_limit, increment_causer_counter) | MCM cause_max_per_causer |
| Sweep rate | credit accumulator, 1 credit = 1 fire | global sweep production | 5 / min | ap_core_callbacks (reads cfg live) | MCM alife_rate |
| Sweep level mix | Bresenham lane pick per credit (current level vs background) | sweep production split | +8 (~4:1 current level) | ap_core_callbacks (reads cfg live) | MCM alife_ratio |
| Sweep refractory | per-squad os.clock spacing | per squad | 60s | ap_core_callbacks | constant |
| BUDGET peek | non-consuming read of global radiant counter | radiant EVAL admission | 5 / 60s (shared window) | ap_core_producer gate 1 | MCM global_consequence_max_events |
| Reactive PACER | token bucket | per callback type | 1 / sec | ap_core_producer | constant |
| Per-squad MVT / Hull threshold | DTO last_<X>_at + arrival reset | per squad per cause | per cause | cause generator | MCM cause_<X>_threshold |

Per-squad threshold (Hull / MVT). Owned by each cause generator. Reads last_<X>_at from a DTO (_ap_stalker_needs, _ap_mutant_instincts, _ap_squad_opportunities); arrival action resets the timestamp. Hull family (needs, instincts) is per-drive: multiple answers under one drive share the timestamp; any answer firing resets the drive's field. Score = weight * (elapsed/threshold)^2. MVT family (stash, area) is per-cause: each cause has its own threshold and timestamp. Gate is binary: elapsed > threshold.

Per-category cause budget is consumed inside the cause generator (self-gating). The producer does not gate cause budgets. A rate-blocked generator is skipped by the EVAL cascade so the slot frees for the next entry. ap_core_limiter.create_cooldown is a small helper for arbitrary timestamp-based cooldowns.

Offmap balance counter. Backs the offmap cause family (SUPPLY_TRADER_OFFMAP, SOCIAL_OFFMAP, JOB_EXPLORE_OFFMAP). Single counter keyed by source level_id; faction-agnostic and destination-agnostic; shared across all offmap causes. Window OFFMAP_WINDOW_SEC (48 game-hours, hardcoded) mirrors the framework pattern where pipeline windows are const and only caps are MCM-exposed (cause_max_offmap, default 2). Clock is xtime.game_sec; bucket state round-trips through save/load via xttltable export / import in ap_core_limiter SAVE_STATE / LOAD_STATE. Off-map filters are prop-only (xsmart.is_base, _is_unclaimed) because smart.stalker_jobs is nil for off-actor-level smarts (smart_terrain.script:462 load_jobs early-exits when not on actor level); has_animated_stalker_jobs / has_stalker_jobs / has_anomaly cannot be used on off-level candidates.

Per-causer reaction counter. A second balance-family counter, parallel to the per-category REACTIONS budget. A causer is the source entity whose action fired a reactive event: the killer (massacre, basekill, squadkill, alphakill), the patient who healed (wounded), or the artefact taker (harvest). For an actor-triggered event the causer id is xconst.ACTOR_ENTITY_ID. Six of the seven reactive causes - all the ones that dispatch a squad at the causer - call check_causer_rate_limit(causer_id) before publishing and increment_causer_counter(causer_id) at publish. The alpha cause is deliberately excluded: alpha_promote only updates the alpha tracker (no squad dispatch), so it neither consumes the per-causer budget nor is blocked by it; gating it would let a promotion silence real reactions and vice versa, and it never keys on the actor anyway (mutant killer only). The counter is keyed by causer id alone (not by cause), so it is shared: once a single source has triggered cause_max_per_causer reactions of any kind within CAUSER_WINDOW_SEC (1 game-hour, hardcoded), every reactive cause attributed to that id is blocked until the window slides. This prevents one dominant source from saturating the reaction layer - the motivating case is a hostile map where the player is the only enemy (Radar), but it also smooths a lone predator clearing a lair or one squad on a long killing spree. Nil causer (unidentified killer) is always allowed; a cooldown cannot key on it. Clock is xtime.game_sec (so the window tracks game time across sleep / time-skip), but the counter is ephemeral - unlike the offmap counter it is NOT persisted; it resets on save load and on level transition (Lua VM reinit). A short throttle does not need to survive a reload. Default cap 1 (MCM cause_max_per_causer) makes the gate a strict one-reaction-per-source-per-game-hour cooldown; raising it admits more.

The world tab in MCM houses the balance family. Pipeline family caps live under framework. Reset button moved to the development tab.

### Tracing

Hierarchical via observe() (consequences, internal phases) and prof + trace:push + debug pattern (cause generators) in ap_core_debug. Each trace carries a monotonic tid (trace ID) and a slash-separated path (span hierarchy). One tid links a cause through its consequence chain into individual actions:

```
[NEEDS] [tid=42 path=needs] FAILED_RULES:WRONG_PERIOD sq=1337 [0.05ms]
[CAUSE.HUNGER_CAMPFIRE] [tid=42 path=needs/cause:hunger_campfire] sq=1337 drive=hunger weight=4.2 [0.15ms]
[CONSEQUENCE.HUNGER_CAMPFIRE] [tid=42 path=needs/cause:hunger_campfire/CONSEQUENCE.HUNGER_CAMPFIRE] success count=1 [0.83ms]
[CONSEQUENCE_PHASE.FIND_DESTINATION] [tid=42 path=needs/cause:hunger_campfire/CONSEQUENCE.HUNGER_CAMPFIRE/CONSEQUENCE_PHASE.FIND_DESTINATION] ok id=445 [0.12ms]
```

Path root is the registered name. Single-cause reactives register the specific cause (cause:massacre etc.) so the root IS the specific cause. Multi-cause radiants register the family string (stash, area, needs, instincts); generators have to run alignment / scan / pick code before a specific cause is known, so the root names the family and the picked cause nests one level under. First line in the example shows a pre-pick rejection (no cause picked, logs under family); subsequent lines show the picked-cause chain.

bracket(constant) in ap_core_debug composes log labels by uppercasing and replacing : with .: "cause:hunger_campfire" -> "[CAUSE.HUNGER_CAMPFIRE]". Each cause/consequence file caches its bracket strings at module load. No hardcoded [CAUSE.X] literals.

Below DEBUG: observe() is a bare passthrough (calls the function, returns the result). trace() returns a null singleton. xprofiler.new_if(false) returns a null singleton. Cost: one enabled() check (~150ns) per call. All null singletons pre-allocated; no allocation at non-debug levels.

---

## Initialization

Four phases. Each requires the previous to complete.

### Phase 0: module load

Engine auto-load resolves .script files on first namespace access (script_engine.cpp:375 sets _G.__index = auto_load). axr_main.on_game_start() iterates .script files alphabetically and triggers auto-load. Module-level code runs immediately on first reference: engine globals cached to locals, constant tables built, xlibs API references captured. No game state at this point - no actor, no entities, no callbacks active.

### Phase 1: on_game_start

axr_main calls on_game_start() on every loaded script. File order alphabetical, but all operations are independent - no cross-module reads at this phase:

- _ap_deps asserts xlibs compatibility. Hard crash on mismatch.
- ap_core_mcm loads config from defaults, registers on_option_change.
- ap_core_debug registers actor_on_first_update for deferred log level init.
- ap_core_callbacks declares both event names at module load (`AddScriptCallback`, phase 0) and installs both detections in its on_game_start: the `ap_squad_on_change` sweep (server_entity_on_unregister eviction + 1s tick, reading cfg.alife_rate/alife_ratio live) and the `ap_npc_medkit_use` hook on `xr_eat_medkit.consume_medkit`.
- ap_core_producer resets dispatch state and registers actor_on_first_update; it subscribes to ap_squad_on_change (via RADIANT_CALLBACKS) but no longer starts or feeds the sweep. register() is write-through (maintains _handlers, _radiant_callbacks, _cascade_buf synchronously); engine subscription is deferred to actor_on_first_update.
- ap_ext_cause_wounded registers its reactive generators against WOUNDED_CALLBACKS; it is a pure subscriber (detection lives in ap_core_callbacks).
- ap_core_consumer registers actor_on_first_update. Does NOT subscribe to xbus yet.
- ap_core_broker registers save/load callbacks, creates the 20s scripted-squad scan timer.
- ap_core_hud resets statistics, registers first_update / option_change / net_destroy / GUI callbacks.
- ap_core_map registers first_update (marker init) + server_entity_on_unregister (marker cleanup on entity death).
- ap_core_compat registers load_state for save cleanup, registers ownership proxy (warfare).
- ap_ext_cause_* / ap_ext_causes_* register generators with producer via register().
- ap_ext_consequences_* register handlers with consumer via register(). Arrival handlers via consumer opts.
- ap_ext_news registers compose timer.

After phase 1: all generators and handlers are registered, _cascade_buf is fully populated (write-through from register()). No engine callbacks subscribed yet, no xbus subscriptions active. Subscription is deferred to actor_on_first_update so predicates do not run before the actor and supporting systems are live.

### Phase 2: actor_on_first_update

Game world and actor exist. Deferred init runs:

- Producer subscribes callbacks (radiant: shared _on_radiant on ap_squad_on_change; reactive: per-callback dispatcher on engine callbacks) and sets _first_update_done. Late register() calls (post-first_update) subscribe new callbacks synchronously.
- Consumer subscribes to xbus cause events.
- Debug reads MCM log level (config now loaded), dumps MCM at DEBUG.
- HUD clears stale map markers from previous session, starts marker timer if MCM map_markers is enabled. Activates statistics overlay if MCM statistics_position is not "off".

actor_on_first_update fires on the first frame with a live actor. It fires on every level transition (not just game start), so handlers use a _subscribed guard to prevent duplicate registration. Named function references registered via RegisterScriptCallback are naturally deduplicated; the guard prevents redundant work.

### Phase 3: on_game_load

Fires after STATE_Read rebuilds all server entities from the save file. At this point alife_object(id) works and entity mutation is safe. Smart mutator re-applies conquered and infested smart data from _conquered_pending and _infested_pending (two-phase restore: load_state reads the data, on_game_load applies the mutations, because load_state fires before entities exist).

Engine load sequence:

1. load_state - save data read (m_data available). Entities do NOT exist yet. `alife_object(id)` returns nil; `level.get_start_time` AVs.
2. STATE_Read - engine deserializes server entities, rebuilds smart terrain config from LTX.
3. on_game_load - entities exist; mutations safe to apply.

Migrations that need to clear engine state (e.g. `scripted_target` on orphan squads) must split into a phase A queue at load_state and a phase B drain at on_game_load. See `doc/standards/code-standards.md` Save Data Migrations for the pattern; `ap_core_compat.script` and `ap_ext_smart_mutator.script` are the reference implementations.

---

## Engine integration

NPC behavior is produced by a four-layer engine chain. AP routes squads to destinations, reads the chain's output, and stays outside the chain itself.

### The four layers

| Layer | Subsystem | What it produces |
|-------|-----------|------------------|
| Content | smartcovers, patrol paths, animpoints (level .ltx) | the animation primitives a job can run |
| Binding | gulag (smart_terrain.script + gulag_general.script) | npc_info[id].job per NPC, scored by priority + precondition |
| Scheme | xr_logic + scheme modules (xr_walker, xr_camper, xr_sleeper, xr_animpoint, xr_smartcover, sr_*) | actions and evaluators registered on the NPC's motivation_action_manager |
| Execution | GOAP action planner (action_planner_script.cpp) | the per-tick action selected against goal world state |

### Write surface

A single engine field: `squad.scripted_target` via `xsquad.acquire_squad` / `release_squad` / `reassert_target`. AP does not write to `npc_info`, `npc_by_job_section`, `job_info_by_job_type_id`, `db.storage[id].active_section` / `active_scheme` / `pstor`, `motivation_action_manager`, or any smart's `stalker_jobs` / `monster_jobs` tables. AP does not call `xr_logic.activate_by_section` or any scheme's `set_scheme` directly. AP does not push exclusive jobs.

### Read surface

| Source | xlibs wrapper | Reader |
|--------|---------------|--------|
| `smart.stalker_jobs` / `monster_jobs` | `xsmart.has_animated_stalker_jobs` | scan-time predicate per cause: reject stub-only smarts (job_type_id in {0, 1}) |
| `smart.npc_info[id].job` | `xsmart.has_jobs_for` | arrival + mid-hold predicate: detect engine binding failure |
| `smart.props` | `xsmart.accepts_faction` | scan-time faction gate (engine target_precondition Tier 1; covers stalker + mutant) |
| `smart.faction` | direct field read | at-base detection (basekill / massacre / squadkill / wounded: `xsmart.is_base(smart) and smart.faction == squad.player_id`); engine sticky setter (smart_terrain.script:1209-1236) |
| `SIMBOARD.smarts[id].squads` | `xsmart.iter_stationed_squads` / `has_squad_of_faction` / `has_enemy_squad` / `is_smart_empty` | physical-presence + hostility checks via is_stationed filter (current_action=1). Cap 5 safety belt. Cross-level safe. Drops sim-intent in-transit squads |

Cause-side filters live in `ap_ext_causes_*.script`; each cause picks the predicate set that matches the activity (surge shelter, lair, base/unclaimed, has-anomaly, etc.).

### Binding race

Engine `select_npc_job` (smart_terrain.script:626-798) can fail to assign a job under two conditions:
1. Full allocation. Every `stalker_jobs` entry is held by `npc_by_job_section` or rejected by `job_avail_to_npc`. `setup_logic` unregisters + re-registers the NPC; engine default idle pose loops.
2. Precondition flip during the post-arrival hold. surge start/end (jobs 2/8/14/19), day↔night for sleeper (3), zombie state, `has_items_to_sell` for trader (15), `has_tech_items` for mechanic (16).

AP detects both via `xsmart.has_jobs_for` and releases the squad cleanly. Mechanism in Squad Lifecycle → Scripted-squad scan steps 4 (arrival) and 6 (mid-hold). Released squads return to SIMBOARD autonomous targeting via `xsquad.release_squad` clearing `scripted_target`.

### SIMBOARD bookkeeping

AP-routed transitions update `SIMBOARD:assign_squad_to_smart` at two hooks: dispatch (`script_squad`, clearing the source roster with `nil` target before engine `sim_squad_scripted:specific_update` bumps `squad.smart_id` to the new target) and commit (`_commit_arrival`, adding the destination roster entry after `xsmart.has_jobs_for` accepts). `SIMBOARD.smarts[id].squads` therefore reflects actual squad placement for AP-routed squads. `has_squad_of_faction`, garrison floor, and faction-quota predicates all read truth. On despawn the engine's own `sim_squad_scripted:remove_squad` clears the roster (`SIMBOARD:assign_squad_to_smart(self, nil)`). Vanilla's own scripted re-homes (via the roster-blind `assign_smart`) are corrected globally by `ap_core_anomaly_fixes` (see Vanilla fixes).

### Vanilla fixes

`ap_core_anomaly_fixes` installs bugfix monkeypatches on vanilla Anomaly A-Life at `on_game_start`, each wrapping (never replacing) the original so it composes with any modpack overlay regardless of load order (see script-loading.md "Composing Wrappers"). A `_patched` flag makes install idempotent.

Roster desync. Vanilla `sim_squad_scripted:specific_update` (script:289) and `:generic_update` (script:356) re-home a scripted squad through `self:assign_smart`, which sets `squad.smart_id` and re-registers NPC jobs but never updates `SIMBOARD.smarts[id].squads` / population. Any squad the engine re-targets this way -- AP's or vanilla's own -- leaves the roster undercounting, so `xsmart.has_capacity` / `is_smart_empty` read wrong. The fix wraps both methods: capture `smart_id` before the original, then call `xsmart.reconcile_squad_roster` after. The reconcile is table-state gated and therefore idempotent -- it corrects a roster-blind base (vanilla, Zona, EFP) and is a silent no-op on a base that already syncs (GAMMA's overlay). Only the SIMBOARD roster table is touched; job binding is left to the original.

The chase override (`ap_core_chase`) also monkeypatches `sim_squad_scripted`, but to implement a feature (squad-target homing), not to fix a bug -- it patches different methods (`get_current_task` / `get_script_target`), so the two coexist without ordering concerns.

### Off-map transit

A cause can flag a destination as off-map. The flag changes selection rules and rate-limiting. The lifecycle machinery is shared with on-map dispatch.

The engine handles the cross-level move through its own per-squad routing, including the offline/online transition machinery for the level swap. At the destination the gulag binds jobs identically to on-map arrivals.

An off-map dispatch has no return leg. The squad travels to the destination, holds the gulag for the shortened off-map `PRE_RELEASE_GULAG` (`SUPPLY_TRADER_OFFMAP` / `JOB_EXPLORE_OFFMAP` / `SOCIAL_OFFMAP`, 600 game-sec = 10 in-zone minutes), then `_unscript_squad` settles it at the destination as an ordinary vanilla resident. The squad does not go back to where it came from; its bond becomes the destination smart. A squad that cannot bind a job at the destination (or loses one mid-hold) is released via `_release_to_dwell`, which keeps only `offmap` + `dispatched_at` so the despawn safety net reclaims it once offline.

AlifePlus stacks safety layers on top of the engine capability. A per-source-level rate counter caps off-map dispatches over a sliding window, so no single level depopulates through repeated outflow. Adjacency narrowing restricts candidate smarts to BFS-reachable neighbor levels, with source level excluded. Cross-level filtering runs only on the data the engine still exposes for off-actor-level smarts. A per-session despawn budget (`offmap_despawn_hours`, default 168 game-hours) reclaims a squad that never settled, offline-only and respecting registered owners + permanent / active-role / task-target protections. SIMBOARD bookkeeping stays current via the dispatch + commit hook pair, so cross-level capacity and garrison queries return accurate counts.

| Layer | Mechanism | Site |
|-------|-----------|------|
| Source-level rate | TTL counter (game-sec, persisted), cap `cfg.cause_max_offmap` (default 2), window `OFFMAP_WINDOW_SEC` (48 game-hours) | `ap_core_limiter.script:110-135` |
| Adjacency narrowing | filter narrows to BFS neighbor set; source level excluded by `xlevel.get_neighbor_levels` removing `source_id` from visited; hop count from `_resolve_offmap_hops` (X-16 + Brain Scorcher + master rank) | `ap_ext_causes_needs.script:151-166`, `xlevel.script:273-287` |
| Cross-level filter | prop-only predicates (`xsmart.is_base`, `_is_unclaimed`); `has_animated_stalker_jobs` omitted because `stalker_jobs` is nil for off-actor-level smarts (`smart_terrain.script:462`) | `ap_ext_causes_needs.script:290-334` (offmap CAUSES entries) |
| Destination selection | `xsmart.find_first_smart` over the narrowed neighbor set (distance-free; foreign-level candidates share no comparable position frame) | `xsmart.script`, via `ap_core_util.find_first_smart_observed` at `ap_ext_causes_needs.script:167` |
| SIMBOARD bookkeeping | `SIMBOARD:assign_squad_to_smart` called at dispatch (source clear via nil target) and commit (destination add), so cross-level capacity / garrison / faction-quota queries read truth | `ap_core_broker.script` `script_squad` + `_commit_arrival` |
| Settle terminal | after the gulag hold, `_unscript_squad` drops the AP entry and leaves the squad at the destination under vanilla AI | `ap_core_broker.script` `_update_gulag` |
| Despawn safety net | offmap entries skip the generic `SCRIPTED_SQUAD_TTL`; `_check_offmap_despawn` reclaims a squad that never settled at `cfg.offmap_despawn_hours` (offline only, respects owner / permanent / active-role / task-target) | `ap_core_broker.script` `_check_offmap_despawn` |
| Arrival check | shared `_commit_arrival` (`has_anchored_jobs` wrapping `xsmart.has_jobs_for`); check short-circuits for off-actor-level smarts so the gulag hold runs to expiry | `ap_core_broker.script` `_commit_arrival` |

Save persistence: the offmap counter exports / imports via xttltable in `ap_core_limiter` SAVE_STATE / LOAD_STATE. The 48-hour window survives save/load and time-skip (game-time clock).

---

## Item flow

Five AP flows touch items: Supply Trader (buy/sell on arrival), Stash Loot (squad takes from stash), Stash Fill (squad deposits surplus to stash), Corpse Loot Trim (an NPC keeps only a policy-bounded share of what it just looted from a body), and Faction Market (a faction's hub traders restock a bounded, rank-gated echo of what that faction's stalkers recently sold). All five load their LTX via `xinventory.load_policy`, classify inventory via `xinventory.get_category(item, opts)` / `xinventory.get_section_category(sec)`, and resolve untouchables through the same categorizer. The first four move items between carriers on squad arrival. The market is the one read-side flow: it observes what the trade flow already did and re-materialises a slice of it for the player.

The unified category model is the modernization step over Alundaio's `items\trade\gulag_job_trade_buy_sell.ltx` (per-section keep counts inside each tradeable item's LTX). Per-section meant every tradeable section had to declare its own buy_sell row, and modpacks adding items had to dirty the item files to participate. AP's `xinventory.get_category` reads engine-baked `_ITM` Parse_ITM buckets (outfit, helmet, artefact, device, money, grenade_ammo, ammo, eatable filtered to food/drink) plus hand-set predicates for the medical 5 (medkit, bandage, antirad, stim, pill) and grenades. A modpack medkit section gets `medkit` category automatically when the engine binds it to `kind=i_medical`. No per-modpack LTX maintenance, no item file edits — the policy operates on categories, the engine assigns categories to sections.

Runtime untouchable checks layered in 1.7.5 (`xinventory.get_category:632-637`): runtime `story_id` via `get_object_story_id` (vanilla quest-tracker convention, `release_npc_inventory.script:81` precedent), `axr_companions.is_assigned_item(opts.npc_id, item:id())` (companion-gifted, `axr_companions.script:1300,1360` precedent), `se_load_var(item_id, "", "strapped_item")` (player-strapped, `itms_manager.script:338` precedent). Section-level untouchable (quest / anim / blacklisted) gates first via `get_section_category`. Equipped check uses `opts.equipped_ids`. All three runtime checks centralized in `get_category` so every consumer (trade, stash loot, stash fill, corpse loot trim, AlifeBalance inventory-balance) respects them without per-consumer code.

`xinventory.load_policy(path, sections, specials_set)` is the shared LTX kernel. Each named section yields `{ entries, rules, specials }`. `entries` preserves declaration order (= BUY / LOOT priority); `rules` is the per-category hash for O(1) lookup; `specials` carries numeric keys named in `specials_set` (`profit_max`, `extras_max`, `fill_max`, ...). Consumer mods ship their own policy LTX under `configs/<mod>/<purpose>_policy.ltx`; the shape is fixed in xlibs, the values are owned by the mod. AlifePlus uses `ap_trade_policy.ltx` for trade, `ap_stash_policy.ltx` for stash, `ap_loot_policy.ltx` for corpse trim, and `ap_market_policy.ltx` for the faction market. AlifeBalance uses `ab_inventory_policy.ltx` for the inventory-balance cull pass.

Per-flow policy lives in dedicated LTX files under `configs/alifeplus/`, DLTX-overridable.

| Flow | Site | Policy | Behavior |
|------|------|--------|----------|
| Supply Trader | `ap_ext_consequences_needs.script` `_arrive_trade` -> `ap_ext_trade.trade` | `ap_trade_policy.ltx` blocks `[ap_trade_policy_rookie]` / `[ap_trade_policy_veteran]` split by `npc:character_rank() >= RANK_VETERAN` (12000) | Four phases: (1) PHASE 1 count NPC inventory per category. (2) PHASE 2 SELL iterates inventory and drops anything whose category count > max; transfers item to seller, credits NPC `floor(cost * 0.5)`; `iterate_surplus` mutates counts in place so PHASE 3 sees the post-sell totals. (3) PHASE 3 BUY walks policy entries in declaration order, fills any category whose count < min using post-sell cash: ammo via `_buy_ammo_tier` (per equipped slot's k_ap-sorted tier map, cheapest section in target tier), consumables via `_buy_consumable_category` (cheapest-first walk over `xinventory.get_category_sections`). (4) PHASE 4 profit cap: if (cash_after - cash_before) > `profit_max`, the excess transfers back to the seller. Only money path. SELL precedes BUY so the cash gained from selling spares funds the buy pass, not just the NPC's walked-in cash. |
| Stash Loot | `ap_ext_consequences_stash.script` `_arrive_loot` | `ap_stash_policy.ltx` block `[ap_stash_policy]` (single uniform block, no per-rank) | Three gates: untouchable in stash -> skip whole stash (no loot, no clear). Crafting in stash AND MCM `consequence_stash_loot_skip_crafting` -> skip whole stash. Otherwise: per-category loot pass (any entry with min > 0, take from stash up to min). Non-ammo entries use section-only resolution and round-robin assignment. Ammo entries use per-member resolution (`xinventory.resolve_ammo_category`) to find a squad member whose equipped pistol or rifle takes the round; the matched member gets the loot. Extras pass takes top-N by `cost` from leftover stash sections (N = `extras_max`). Always clears the stash at end. Sections the squad does not take are destroyed by the clear. |
| Stash Fill | `ap_ext_consequences_stash.script` `_arrive_fill` | same `ap_stash_policy.ltx` block | Deposits any category whose squad count exceeds policy max, total per-event capped by `fill_max`. Picks items from online squad members' inventories using per-member `get_category_opts` for ammo tier resolution. `treasure_manager` round-trips sections as section names only; stateful categories (weapon, outfit, helmet, device, ammo with non-default count) survive only the section name. |
| Corpse Loot Trim | `ap_ext_loot_select.script` `_trim_npc_deferred` (reactive `npc_on_get_all_from_corpse`, deferred one frame via `CreateTimeEvent`) | `ap_loot_policy.ltx` block `[ap_loot_policy]` (single uniform block; loot uses `max` only, `min` ignored) | Records the item ids an NPC takes from a corpse (only when the engine will transfer them, lootable flag non-nil), then next frame classifies the looter's whole inventory and, per capped policy key, releases the cheapest looted members first (`get_cost` ascending) via `release_item` until the count is at/under max. Acts only on the just-looted set, so the kept loot is the highest-value and any standing excess is left to the AlifeBalance cull. Ammo-for-weapon is automatic via `get_category_opts` (equipped pistol/rifle tiers kept; `ammo_not_equipped` capped to delete). Online only; skips unscriptable looters (companions, story, traders, named). No money path, no save state. |

The fifth flow, the faction market, is not a squad-arrival move and has its own pipeline below.

`ap_ext_consequences_needs.script` `_on_arrive` previously also ran a per-need item-consumption step (HUNGER ate food, HEAL used medkits, etc.); dropped per n477. Need satisfaction is now the arrival itself (DTO timestamp reset). The Supply Trader arrival action above is the only item-mutation flow under the Needs umbrella.

Money moves only inside `ap_ext_trade.trade` (via `xcreature.transfer_money` and `xcreature.give_money`, both defensive against the engine `TransferMoney` u32 underflow at `script_game_object_inventory_owner.cpp:693`). See todo-demonized-exes.md n015 for the engine PR. The faction market moves no money of its own: it adds stock to a trader and overrides the engine's *displayed* price for tagged goods (`on_get_item_cost`); the player pays that premium through the normal engine buy, the same path as any other trader purchase.

### Faction market (ap_ext_market)

The market is the read side of the trade flow. It moves no squad and no money. It watches what the trade flow already did, remembers it per faction, and re-stocks a faction's own traders with a bounded echo for the player to buy.

```
ap_ext_trade.trade (an NPC sells at a trader)
  -> observe (xevent hook on ap_ext_trade.trade)
       record the sold sections under the seller's faction
         -> ledger: per-faction bounded dedup FIFO of section names

engine restock (a trader respawns its stock)
  -> detect (Destockifier trader_on_restock callback, else trade_manager.update hook)
       -> inject: filter the trader's faction ledger, spawn the survivors into the trader
            -> price (on_get_item_cost): tagged goods cost full base x the MCM premium
                 -> release (timer): drop tagged goods past the window
```

The write side and the read side never touch each other directly. They meet only through the ledger.

Principles:

- Observe, never modify. The trade flow is read through a function hook on `ap_ext_trade.trade`; that function is unchanged. The market is a consumer of trade output, not a participant in trade.
- Echo, not conservation. Every restock purges the trader's ruck (`purchase_list.cpp:20`), so the item a stalker sold is already gone. The market re-creates a bounded echo of the sale, never the original, so nothing is duplicated.
- Different stock per trader. A section injected at one trader is claimed for the goods window, so a sibling faction trader restocking inside that window draws other sections.
- Player-facing scarcity. One section, one trader, one short window, gated by the player's rank, at a premium. The gate evaluates the player, never the NPC.
- No money path. The market adds stock and overrides the engine's displayed price for tagged goods; the player pays through the normal engine buy.

Ledger. Per faction, a bounded dedup FIFO of section names. A re-sold section moves to the newest end; the oldest is evicted past the cap. The section is the only thing kept, since injection re-derives cost, category, and rank from it. It is a plain table saved and loaded verbatim, so it carries no clock and is safe at `load_state`, where `game_sec` would AV.

Persistence. Both halves of the market state survive save/load: the ledger and the tagged-goods table (`item_id -> { trader_id, inject_time }`). The injected items are real server entities that the save already preserves; persisting the tag table alongside the ledger keeps their premium price and timed release across a reload, since a player reloads inside the goods window as a matter of course. At `load_state` only the tables are restored (no alife queries, which AV before entities exist); at first update a validation pass drops any tag whose item the engine purged on an offline restock between save and load. A recycled id that survives validation is rejected at pricing time by the `parent == trader_id` guard in the cost hook, so premium never mis-applies.

Inject filter. A ledger section reaches a trader's stock only if it survives, in order: the always-excluded set (bugged sections such as `grenade_smoke`), the distribution claim (unclaimed this window), the policy whitelist and its per-category cap, role fit (a medic stocks consumables, a general trader stocks all), and the player-rank roll (`xinventory.get_rank_chance`). The per-trader count is set in MCM (items per trader); the policy file supplies the per-category caps within that total. The whitelist is scoped to categories traders actually display (consumables, artefacts, devices, grenades); gear (outfits, helmets, weapons) is excluded, since traders do not sell it. Premium and spawn condition are MCM settings (Economy > Faction market), not policy specials.

Economy loop. Loot, trade, and market are one cycle. The corpse loot trim bounds what a stalker keeps off a body. The trade flow turns that surplus into cash and restocks the NPC's ammo, the loop the NPCs run for themselves. The market taps that flow and returns a rank-gated, premium slice to the player. The injected stock is bounded by what the faction actually sells, so it never exceeds the gear that faction fields.

---

## Squad Lifecycle

ap_core_broker manages the full lifecycle: scripting, arrival detection, post-arrival wait, release.

### Scripting

script_squad(squad, smart, opts) sets scripted_target via xsquad.acquire_squad. scripted_target routes the squad to specific_update (direct A->B movement). AP clears __lock on acquisition; scripted_target alone is sufficient for routing. If another mod clears scripted_target between ticks, generic_update runs and the squad may be reassigned by SIMBOARD; reassert_target restores scripted_target within 20s. The squad is registered in _ap_scripted_squads with TTL, optional arrival handler, and wait duration.

script_actor_target(squad) scripts a squad to pursue the player using engine-native actor targeting (no arrival detection).

script_squad_target(squad, target_id, opts) scripts a squad to pursue another squad by id via continuous coordinate homing (the ap_core_chase hooks; see Chase pattern). No arrival detection - the chase ends when the target dies or the leash releases. Both chase forms set entry.is_chase and run the evaluate_chase leash on the periodic scan instead of the arrival/gulag flow.

### Scripted-squad scan

_update_scripted_squads runs every 20s via CreateTimeEvent. For each tracked squad:

1. Entity check. xobject.se(squad_id). Gone -> remove from tracking. Catches squads that died or despawned between scans.
2. Reassert. xsquad.reassert_target(squad, data.scripted_target). Restores scripted_target if another mod overwrote it; clears __lock. Vintar-class mods set these fields every tick on their own squads; AP reasserts every 20s on its squads.
3. TTL. 7200 game-seconds. Expired -> unscript. Prevents permanently pinned squads.
4. Arrival. xsmart.is_arrived(squad, smart). On arrival, dispatch the registered on_arrive function; then check engine job assignment via xsmart.has_jobs_for. If smart is online + on actor's level and any member has no job, unscript (release_no_jobs). If smart is offline or off-level, AP cannot observe job state and enters the wait state by default. This is the runtime-allocation half of the dispatch-viability check. The cause-side filter is the scan-time half: xsmart.has_animated_stalker_jobs excludes stub-only smarts at find_smart time so the dispatch never happens on structurally barren on-level targets; xsmart.has_jobs_for releases the squad if real slots ended up taken between dispatch and arrival.
5. Wait. release_at = game_sec() + pre_release_gulag (default 300s). When game time exceeds release_at, unscript. Game time advances during sleep / time-skip and survives save/load.
6. Mid-hold re-check. While the squad is in its pre_release_gulag wait, _update_pre_release_gulag re-runs xsmart.has_jobs_for every 20s (same is_on_actor_level gate as step 4). The engine may re-run select_npc_job during the wait on a precondition flip (surge transition, day/night flip for sleepers, zombie state, trader has_items_to_sell). If no eligible replacement job exists, npc_info[id].job becomes nil and the setup_logic freeze loop reactivates; the re-check unscripts early with release_no_jobs_mid_hold so the squad does not stand idle at a smart that no longer accepts it. For off-level smarts the gate short-circuits and the wait runs to expiry.

### Activity record

ap_core_broker owns a 256-slot FIFO of dispatched consequences. Each record entry is one consequence dispatch on one squad. The substrate is shared by HUD markers, ap_ext_news compose, and external integrators (warfare).

Write. ap_core_record.add_record(subject_squad, cause_key, consequence_key, opts) is the single entry point; consequence handlers call it after script_squad / script_actor_target SUCCESS. add_record captures display keys via _capture_side for subject + optional other (engine community string for stalker / zombified-stalker squads → faction_key; "st_ap_macros_species_" + xcreature.get_mutant_species(squad) for mutant squads → species_key; xsquad.get_commander_name → name) and writes the entry. Squad-derived keys are eager so dead/unregistered squads remain renderable. Smart, level, and game-hour-at-write stay lazy ids resolved at render.

Substrate. _ap_record (xttltable.fifo, capacity 256, on_evict callback) holds entries keyed by monotonic cons_id. _record_assigned[squad_id] indexes the live entry per squad. _record_seq is the cons_id counter. Writing a new entry for a squad flips the previous entry's assigned flag to false before assigning the new one. _on_record_evict nulls _record_assigned only if the evicted entry was the live one (a flipped predecessor leaves no index entry to clean).

Lifecycle. broker registers SERVER_ENTITY_ON_UNREGISTER and clears the record's assigned flag on entity death. HUD has its own SERVER_ENTITY_ON_UNREGISTER for marker cleanup; the two callbacks fire independently. Records persist beyond unscripting (the squad may be released but the record stays for query) until either the squad dies (clear_record) or the FIFO evicts.

Query API (ap_core_broker, mirrored on ap_api):

| call | purpose |
|------|---------|
| record(subject_squad_id, cause, consequence, opts) | write entry, return cons_id |
| get_record(opts) | most-recent entry matching filter (highest cons_id); { squad_id, assigned=true } hits the live index in O(1) |
| get_records(opts) | all entries matching filter; assigned=true reads index, else full FIFO scan |
| clear_record(squad_id) | flip assigned to false (called by SERVER_ENTITY_ON_UNREGISTER) |

Entry schema (suffix convention: `_id` for ids, `_key` for translation keys, names are engine strings):
- ids: `cons_id`, `squad_id`, `cause_id`, `other_squad_id`, `smart_id`, `level_id`
- translation keys: `cause_key`, `consequence_key`, `action_key`, `subject_faction_key`, `subject_species_key`, `other_faction_key`, `other_species_key`
- engine strings: `subject_name`, `other_name`
- state/value: `assigned` (bool), `game_hours` (number)

Translation keys are directly translatable via `game.translate_string`. Capture is engine-only: `xcreature.get_mutant_species(squad)` non-nil routes to `subject_species_key = "st_ap_macros_species_" .. species`; otherwise `subject_faction_key = squad.player_id` (vanilla XML covers stalker / zombified-stalker community strings). `subject_name = xsquad.get_commander_name(squad)`. Same rule for the optional other side.

### Save / load

_ap_scripted_squads + ap_record_entries (walked from FIFO via :each into a sequential array) + ap_record_seq persist to m_data.ap_core_broker. Off-map session fields live on the scripted-squad entries themselves; there is no separate offmap save key. Engine-side scripted_target persists natively across save/load (sim_squad_scripted STATE_Write / STATE_Read). Arrival handler functions are transient - they are re-registered every load via consumer.register opts (the consumer wires on_arrive opts to broker.register_arrival_handler). On load, squads marked as arrived get release_at = 0 (immediate release on next scan); the FIFO rebuilds from the saved array via :set on each entry, and _record_assigned rebuilds from entries with assigned == true.

### Off-map track

Off-map sessions live on the same `_ap_scripted_squads` entries as regular scripted dispatches; the broker has one collection, one scan, one source of truth. An off-map dispatch is a regular `script_squad(squad, smart, { offmap = true })` call that additionally initialises the session fields `{ offmap = true, dispatched_at }` via `_init_offmap_session`. There is no return leg: the squad travels out, holds the gulag briefly, and settles at the destination. The 20s scripted-squad scan calls `_scan_offmap_entry` for any entry with `data.offmap`, which runs only the despawn safety-net check.

Registration. `_init_offmap_session(data, squad_id, now)` sets `offmap = true` and `dispatched_at = now` on a fresh `script_squad` entry. No home level is captured -- the squad's origin is not tracked because it never returns to it.

TTL. Off-map sessions have no scripted-squad TTL. The generic TTL exists to release scripts that don't reach their target on a reasonable schedule; an off-map trip can span multiple maps (an online stalker walking through level changers takes many game-hours), so the session lifetime is bounded by `cfg.offmap_despawn_hours` instead, owned by `_check_offmap_despawn`. `_update_scripted_squad` gates the TTL check on `not data.offmap`.

Arrival. Off-map arrival is an ordinary arrival: `_run_arrival` runs the on_arrive handler (the trade / explore / social action) then `_commit_arrival`, which starts the gulag hold (`release_at = now + pre_release_gulag`, the shortened off-map value) and adds the destination roster entry. There is no off-map-specific arrival handling.

Settle. `_update_gulag` releases at `release_at` via `_unscript_squad` for every entry (off-map and on-map alike): the AP entry is dropped and the squad becomes an ordinary vanilla resident of the destination smart. An off-map squad that cannot bind a job at commit, or loses one mid-hold, is released via `_release_to_dwell` instead, which clears the scripted fields but keeps `offmap` + `dispatched_at` so the despawn safety net can reclaim it.

Re-dispatch preservation. `script_squad` preserves the off-map session across a non-offmap re-dispatch: if the existing entry has `offmap = true` and the new dispatch is non-offmap, `_preserve_offmap_session` carries `offmap` + `dispatched_at` into the new entry, so the despawn budget keeps counting from the original dispatch.

Despawn safety net. `_check_offmap_despawn` reclaims a squad that never settled: it fires when `(now - data.dispatched_at) > cfg.offmap_despawn_hours * 3600` (default 168 game-hours = 7 days, MCM-tunable) and `not squad.online`. Online squads may be in transit or under player observation; they don't despawn. `is_protected` runs first so story / task / companion / warfare-owned squads are spared. Release goes through the engine's `sim_squad_scripted:remove_squad`, which clears the SIMBOARD roster on the way out.

Save / load. The merged collection persists to `m_data.ap_core_broker.ap_scripted_squads`; session fields ride along on each entry. `dispatched_at` is an absolute game-second stamp, so it stays correct across save/load and time-skip. `SERVER_ENTITY_ON_UNREGISTER` clears the squad's entry. Two-phase recycled-id sweep: `_on_load_state` restores the dict from m_data (entities aren't loaded yet, no alife queries), and `_on_game_load` sweeps it via `xsquad.is_squad` to evict stale ids once `alife_object(id)` is valid. The engine recycles freed server-entity slots and SERVER_ENTITY_ON_UNREGISTER can be bypassed by some release paths (`alife_object_registry_inline.h:24-34` removes without firing on_unregister), so a saved squad_id may resolve to a different class (script_zone, item) after the original squad was released. The scan-time class check in `_update_scripted_squad` catches in-flight corruption that bypassed unregister mid-session.

Broker-internal `is_offmap_dispatched(squad_id)` returns true when the squad's entry has `offmap = true`; the cause-side guard in `ap_ext_causes_needs.script:152-154` reads this to block off-map causes from re-publishing while the session is active.

### Protection (ap_core_broker.is_protected)

Delegates to xsquad.is_protected with five guard categories (ap_core_broker.script:66-76, _protection_opts):

| Category | Sub-reasons |
|----------|-------------|
| Ownership | exclude_filter = get_owner; reason IS_OWNED. Today: warfare proxy in ap_core_compat |
| Scripted | engine scripted_target set; condlist; random_targets |
| Permanent | story NPC; trader; named NPC; empty squad |
| Active role | task giver; companion |
| Task target | assault, bounty, hostage, delivery, dominance, rescue |

script_squad does not check protection. It assumes upstream layers verified the squad. Direct callers outside the pipeline must check is_protected themselves.

### Reactive preemption (interruptable flag)

Radiant cadence accumulates AP-scripted squads. Reactive events (massacre, wounded, basekill, harvest, squadkill, alphakill) need an unscripted squad to dispatch their consequence; if the pool is exhausted by radiant scripts, reactive starves. Preemption: reactive consequences opt in to a fallback pass over AP's tracked pool, accepting squads marked interruptable.

Interruptable flag. script_squad stores `opts.interruptable` on `_ap_scripted_squads[id].interruptable`. Every consequence dispatch sets the flag explicitly at the call site - there is no pipeline-level default contract:

- `interruptable = true`: on-map needs (14 of 17), all instincts (7), and stash_ambush. Routine maintenance trips, deferrable.
- `interruptable = false`: all reactions (massacre, wounded, basekill, squadkill, harvest, alphakill), the chase dispatch in ap_ext_common (move_actor_chasers, move_squad_chasers), area causes (conquer, swarm, infest), stash_loot, stash_fill, and the 3 off-map needs (supply_trader_offmap, social_offmap, job_explore_offmap). Reactions are in-flight responses; area mutations and stash item-actions have persistent on-arrival side effects worth preserving; off-map dispatches are committed cross-map trips that lose their destination if preempted outbound.

The broker default is `true` only as a safety net; no caller relies on it.

Two-pass find. ap_core_util.find_squads runs the standard protection-opts pass first. If the caller passes `opts.allow_preempt = true` AND the standard pass returns zero candidates, a second pass runs with opts built via `broker.build_preempt_opts`:

- `source = _ap_scripted_squads` (iterate AP's small tracked pool, not SIMBOARD or the per-level cache)
- `exclude_scripted = false` (do not reject scripted; we want interruptable ones)
- `exclude_filter = _preempt_filter` (rejects external owners and squads with scripted_target set but not in _ap_scripted_squads)
- `exclude_ids` = freshly-built set of AP-scripted squad ids where `data.interruptable == false` (keeps non-interruptable AP scripts out of the iteration entirely)

Interruptable AP-scripted squads pass all gates and become candidates. xsquad.find_squads still applies permanent / active_role / task_target via the boolean opts, plus distance / faction / level / exclude_at_smart_id checks. Single pass, no recursive find.

Release-and-rescript. script_squad already handles "already scripted" by calling xsquad.release_squad on the existing target before acquiring the new one (`broker.script_squad:446-452`). The preempt flow piggybacks: when the reactive picker selects an interruptable scripted squad and calls script_squad, the old radiant trip is dropped and the new reactive script replaces it.

Bounded by negation. Reactive does not preempt reactive (reactive scripts carry `interruptable = false`). External scripts (warfare, story, task) are not touched (rejected at gate 1 by exclude_filter). Radiant pipeline is unchanged - it still scripts squads at the same cadence; reactive gets guaranteed access via fallback rather than via radiant throttling.

### Coordination

scripted_target is the squad control field. Setting it routes the squad to specific_update. AP no longer sets __lock (clears it on acquisition). scripted_target alone is sufficient; __lock was a redundant fallback guard. AP checks scripted_target at gate 2 (is_protected, via xsquad.is_scripted). Two alife mods that both check scripted_target before claiming a squad will not conflict.

xsquad.acquire_squad sets scripted_target on acquire; xsquad.release_squad clears it on release; xsquad.reassert_target restores it if overwritten. The ownership registry (broker.register_owner) adds identity on top: it tells AP WHO owns a squad, not just that it's owned. Vintar-class mods reassert every tick on their own squads; AP reasserts every 20s. Warfare is registered by default in ap_core_compat.

---

## PDA map markers (ap_core_map)

ap_core_map owns the marker lifecycle end-to-end via a private _marker_state[squad_id] = { cons_id, remove_at? } table. cons_id is the activity record monotonic id of the entry last rendered (cache miss when a fresh entry is recorded for the same squad). Marker timer fires every UPDATE_MARKERS_SEC = 10s (apply); validate runs on the same timer at VALIDATE_MARKERS_SEC = 20s cadence. Reads from ap_core_record. Owns no domain state.

Apply pass. Iterates ap_core_record.get_records({assigned=true}) and calls _mark_squad on each. _mark_squad short-circuits when state.cons_id matches; otherwise resolves the action label via game.translate_string(entry.action_key) (the ACTION enum value is itself the localization id, e.g. "action:massacre_investigate" -> "Investigating a Massacre Site"). The marker label is a multi-line block built inline from record fields: action / `Subject: <name | translated species> (translated faction)` / optional `Other: ...` line when entry.other_squad_id is non-nil. xpda.mark_squad is called and _marker_state[squad_id] caches the cons_id.

Validate pass. Iterates _marker_state itself. Three-stage chain per squad: get_record({squad_id, assigned=true}) -> xobject.se(squad_id) -> se.scripted_target. If any stage fails, starts a MARKER_LINGER_SEC = 60s linger timer (state.remove_at) with reason `evicted` (record gone), `dead` (server entity gone), or `unscripted` (squad no longer routed). When the linger expires, unmarks via xpda.unmark_squad and drops the slot. Entity death also triggers _on_server_entity_on_unregister (in ap_core_map) which unmarks immediately, no linger.

Subject resolution. The record entry carries translation keys for both subject and other (`subject_faction_key`, `subject_species_key`, `subject_name`, plus the `other_*` mirror). ap_core_map calls game.translate_string on each `_key` at render time and runs an inline `_format_party` helper to compose `<name | translated species> (translated faction)`. No ext import.

ALPHA_PROMOTE flow. Alpha squads write a record via ap_ext_consequences_alpha (calls add_record with CAUSE.ALPHA / CONSEQUENCE.ALPHA_PROMOTE) but never call script_squad. Their record is `assigned=true` so apply renders the marker. validate sees `not se.scripted_target` and starts linger after the next 20s pass; the marker fades MARKER_LINGER_SEC after promotion. Re-promotion (new alpha record) resets the cycle.

Right-click teleport. ap_core_map registers `map_spot_menu_add_property` and `map_spot_menu_property_clicked`. Gate: `_marker_state[id] ~= nil` and `cfg.map_markers`; the menu item appears only on AP-rendered spots and only while markers are enabled. On click, `xobject.se(id).position` (the squad server-object's `o_Position`, synced from commander per `cse_alife_online_offline_group`) is the destination. Same level uses `db.actor:set_actor_position(pos)`; cross-level uses `ChangeLevel(pos, m_level_vertex_id, m_game_vertex_id, VEC_ZERO, true)` after a `m_level_vertex_id < INVALID_LEVEL_VERTEX_ID` guard. No combat / weight / bleed / surge gates; the feature rides on the `map_markers` MCM toggle (development tab) which is itself dev-tier.

## Statistics overlay (ap_core_hud)

Pipeline counters per pipeline (radiant, reactive). Track events, gate denials, cause publishes, consequence results, blocker breakdown. classify(result, is_radiant) routes each consequence result to the appropriate counter. UIStatsHUD renders the ledger in four sections (INCOMING, DENIED, EVALUATED, OUTCOME) with radiant and reactive columns and a sliding-window extra column: per-minute rates and denial percentages computed over the last ~60s of counter deltas (a 7-slot snapshot ring fed by the 10s dump), so the overlay shows now, not the session average. Three colors: aged cream for structure and neutral flow, muted green for OUTCOME values, muted red only on loss rows (the DENIED section and the limiter) and their nonzero percentages; RULES/SCAN rejections stay cream because they are the simulation deciding "not now", so red on screen always means something to look at. Rows: throttle (reactive 1/sec-per-type flood control) and balance (reactive on/off-map mix) under DENIED; the right column carries a "last 60s" header naming its clock. Font letterica16. Six screen positions via MCM. Hides on PDA, inventory, menus. Zero overhead when statistics_position is "off". Log dump every 10s ([STATS] TOTAL/MIN RADIANT and REACTIVE) with the same full-word labels the overlay uses.

---

## News

ap_ext_news transforms AP event telemetry into stalker radio chatter via per-consequence templates in ui_st_ap_news.xml. There is no grammar engine. Presentation layer with one-way dataflow: pipeline emits, news consumes, news never writes back to pipeline state.

### Write path (ap_core_record.add_record)

Consequence SUCCESS calls ap_core_record.add_record(subject_squad, cause_key, consequence_key, opts). _capture_side(subject) and _capture_side(other) read engine fields and produce three display keys per side: faction_key (community string for stalker / zombified-stalker, nil for mutant), species_key ("st_ap_macros_species_" + xcreature.get_mutant_species, nil for stalker), name (xsquad.get_commander_name, nil for mutant). Squad-derived keys are eager. Smart, level, and event-time game hour are stored verbatim from opts (smart resolved lazily at compose time via xobject.se + xlibs resolvers).

Every consequence dispatch produces one record. Pacing is the compose interval + dedup ring; news reads from the broker activity record.

### Drain path (compose tick)

Composer tick fires on an MCM-randomized interval (defaults 60-200s via news_interval_min_sec / _max_sec, slider range 1-600). Each tick:

1. Read ap_core_record.get_records() (full FIFO scan), filter to unreported entries (cons_id not in _reported_cons_ids ring) on the player's current level whose age is within news_max_age_game_hours.
2. Pick one at random (gossip, not sequential log).
3. Pick a per-consequence template from the pool (st_ap_news_tpl_<consequence>_NNN), filter variants by required slots, random pick.
4. Substitute slots.
5. Pick a speaker via _pick_speaker.
6. Dispatch via dynamic_news_manager:PushToChannel (or xpda.send during the cold-start window).
7. Mark cons_id in _reported_cons_ids (capacity 512, FIFO eviction). Same record never narrated twice.

### Slots

| Slot | Source |
|------|--------|
| #subject_faction# | game.translate_string(entry.subject_faction_key) (compose-time) |
| #subject_name# | entry.subject_name (engine string, captured at add_record) |
| #subject_species# | game.translate_string(entry.subject_species_key) (compose-time) |
| #other_faction# | game.translate_string(entry.other_faction_key) |
| #other_name# | entry.other_name |
| #other_species# | game.translate_string(entry.other_species_key) |
| #location# | xlevel.get_smart_display_name(xobject.se(smart_id)) (lazy) |
| #level# | xlevel.get_level_name(level_id) (lazy) |
| #ago# | hours-since game_hours: empty (recent), _ago_recent, or _ago_hours |

_filter_variants drops variants whose required slot is empty before the random pick. _clean_output trims, collapses double spaces, removes orphan punctuation.

### Speaker selection (_pick_speaker)

Iterates db.OnlineStalkers once per tick and filters by:

1. alive, hydrated game_object, IsStalker, not story NPC, not in combat (:best_enemy()).
2. Canon channel restriction: dynamic_news_manager.channel_status[community] must be true. For player's faction this gates Monolith / Army / Greh / ISG to members only.
3. Scope filter (news_scope MCM enum): own (community equals actor's), allies (own OR is_factions_friends), world (any non-enemy).
4. alignment_human (humans only; mutants never speak).

A random NPC is sampled from the filtered pool. Speaker's community becomes the channel: PushToChannel(speaker:character_community(), { Mg, Se, Ic, Snd="news", It="npc" }). No npc:see call (heavy, irrelevant for old gossip), no proximity cascade (events pre-filtered to the current level), no faction routing by subject - the speaker IS the channel.

### Cold-start fallback

When dynamic_news_manager.get_dynamic_news() returns nil (~27s window after game load), _dispatch falls back to xpda.send (direct give_game_news). Documented exception in library/modding/anti-patterns.md - canonical pattern, not the bypass anti-pattern.

### MCM

news_enabled (bool master), news_interval_min_sec / news_interval_max_sec (int 1-600), news_scope (own / allies / world; default allies), news_max_age_game_hours (int 1-72, default 12). The composer randomizes the next delay between min/max bounds after every tick.

### Trace codes (ap_core_const.TRACE)

NONE (disabled or no entries), NO_TEMPLATES, NO_MSG, NO_SENDER, SENT.

### Constants in ap_ext_const

(none for record display; capture is engine-direct in `ap_core_broker._capture_side`).

### Invariants

- Squad-derived translation keys captured eagerly at add_record time. The engine value drives the slot: mutant squads get a species key (`st_ap_macros_species_<x>`), stalker / zombified-stalker squads get the raw community string (vanilla XML resolves). Smart / level / time stay lazy. Death-resilient: dead or unregistered squads remain renderable.
- Storage is the broker activity record. _reported_cons_ids ring (512, FIFO) is session-lifetime; resets on load. Template pools rebuild on locale change.
- Empty slots are nil-safe. Variant filter rejects, or slot substitutes to empty string and _clean_output collapses the gap.
- Channel routing is speaker-driven: speaker's community = channel. No subject-faction channel mapping, no per-consequence routing rules.
- Events pre-filtered to level.current() before random pick. Speaker pool db.OnlineStalkers (overwhelmingly the player's level).
- Templates immutable. Per-tick slot table builds fresh and is discarded after one substitution.
- Locale switches mid-session leave records holding strings in the previous locale until natural eviction. Pragmatic accept.

### Content

ui_st_ap_news.xml ships per locale (eng, rus). Per-consequence templates plus mutant faction and species display plurals plus ago strings. Validator enforces id-set parity between locales.

---

## Domain layer

### Tracker (ap_ext_tracker)

Domain state manager.

| State | Storage | Notes |
|-------|---------|-------|
| Killers | _ap_killers (kill counts per entity) | populated on squad_on_npc_death |
| Alphas | _ap_alphas (level / kills / name) | only mutants become alphas; stalker rank handled natively by engine |
| Alpha-dead grace | _ap_alpha_dead (xttltable TTL, 3600s) | is_alpha returns true during grace |
| Stalker NEEDS DTO | _ap_stalker_needs (per squad, 9 timestamps) | NEED_FIELDS in ap_ext_const, line 281 |
| Mutant INSTINCTS DTO | _ap_mutant_instincts (per squad, 5 timestamps) | INSTINCT_FIELDS in ap_ext_const, line 288 |
| Squad OPPORTUNITY DTO | _ap_squad_opportunities (per squad, 6 timestamps) | OPPORTUNITY_FIELDS in ap_ext_const line 295: stash_loot, stash_ambush, stash_fill, area_conquer, area_swarm, area_infest |

DTO ownership: only the owning generator writes; any cause generator may read (Cross-DTO read pattern). Save/load: m_data.ap_ext_tracker holds killers and alphas only. The three DTOs (NEEDS, INSTINCTS, OPPORTUNITY) are session-only - they reset to empty fifo_caches (capacity 100 each) on game start and on load, populated lazily by cause generators on first squad reference.

ALPHA_PROMOTE marker integration is record-driven: ap_ext_consequences_alpha calls add_record with CONSEQUENCE.ALPHA_PROMOTE on the killer squad after a successful update_alpha. HUD picks the marker up via the standard get_records({assigned=true}) path and lingers it after promotion.

### Smart Mutator (ap_ext_smart_mutator)

Runtime smart terrain mutations: territory conquest (shared spawn) and mutant infestation (exclusive spawn).

#### Engine respawn pathways

Two pathways (smart_terrain.script:246-304, 1657-1700):

Pathway 1: LTX respawn_params (~460 smarts). Most smarts define spawn sections in LTX (e.g. spawn_stalker@advanced with spawn_squads = stalker_sim_squad_novice). These entries have NO .faction field. The respawn filter at line 1667 passes via self.faction_controlled == nil - ALL params fire regardless of who occupies the smart. Natural population baseline.

Pathway 2: faction_controlled (~16 vanilla smarts). Smarts with faction_controlled in LTX generate respawn_params entries with .faction fields. The respawn filter gates spawning: if v.faction == self.faction. Changing self.faction switches which faction respawns. Engine's designed mechanism for dynamic territory control (smart_terrain.script:246-267).

Faction resolution: check_smart_faction (smart_terrain.script:1209-1236) runs every update tick for online smarts. Counts IsStalker NPCs present and sets self.faction. When empty: self.faction = self.default_faction (nil for most smarts). Monsters (IsMonster) are invisible to this function; a smart occupied only by mutants reverts as if empty. Runs AFTER try_respawn in the update cycle (line 1279 vs 1253). Online smarts only.

#### Conquest (shared spawn)

conquer_smart(smart_id, faction) calls xsmart.set_shared_spawn(smart, "ap_conquest", faction, spawn_num). Adds ONE respawn_params entry for the conqueror's faction. The entry has no .faction field and faction_controlled is NOT set. Because faction_controlled stays nil, the engine's respawn filter at line 1667 passes ALL entries unconditionally - both the original LTX entries and the injected conquest entry fire. The conqueror's squads appear alongside the originals, competing for max_population slots. Squad sections come from xsmart.SQUADS_BY_FACTION: stalker factions spawn *_sim_squad_novice / advanced / veteran; mutant factions spawn simulation_* sections.

Coexistence:

- max_population=1 mutant lair: conquest entry competes with the original mutant entry. Engine picks one eligible entry at random per respawn cycle (smart_terrain.script:1707). Sometimes a mutant spawns, sometimes the conqueror's squad.
- max_population=3 stalker camp: the conquest entry adds one squad slot alongside existing stalker spawns. Mixed presence, not replacement.

Revert. xsmart_spawn.clear_shared_spawn(smart, "ap_conquest") removes the entry. Smart returns to its original LTX-only spawn tables. faction_controlled and smart.faction were never modified.

Volatility. Engine rebuilds respawn_params from LTX on every load (STATE_Read calls read_params), so the injected entry is lost. Two-phase restore: load_state -> _conquered_pending; on_game_load applies via set_shared_spawn after entities exist. 60s periodic scanner re-applies injections as a safety net.

Decay. cfg.mutator_area_conquest_decay_hours (default 48). Scanner checks xtime.game_sec() - conquered_at and calls clear_shared_spawn on expired entries. Original LTX spawns never interrupted - decay just removes the extra entry.

Per-level cap. can_conquest_on_level(level_id) counts conquered smarts with matching level_id. Rejects if count >= cfg.mutator_area_conquest_max_per_level (default 3, MCM 1-5). Same-faction re-conquest refreshes the timestamp without consuming a slot. Different-faction overwrite on an existing entry also bypasses the cap.

#### Swarm (shared spawn, mutants)

swarm_smart(smart_id, species) calls xsmart_spawn.set_shared_spawn(smart, "ap_swarm", species, spawn_num). Same mechanism as conquest - additive shared spawn entry, no faction_controlled, no .faction field. The injected entry fires alongside the originals. Species comes from xsmart_spawn.SQUADS_BY_SPECIES (simulation_* sections).

Independence from conquest. _swarmed_smarts is a separate table from _conquered_smarts. Same-smart conquest and swarm coexist as two distinct respawn_params entries (ap_conquest + ap_swarm); engine picks one eligible entry per respawn cycle. Save and load round-trip each table separately.

Decay. cfg.mutator_area_swarm_decay_hours (default 48). Scanner checks xtime.game_sec() - swarmed_at and calls clear_shared_spawn on expired entries.

Per-level cap. can_swarm_on_level(level_id) counts swarmed smarts with matching level_id. Rejects if count >= cfg.mutator_area_swarm_max_per_level (default 3, MCM 1-5). Same-species re-swarm refreshes the timestamp without consuming a slot. Different-species overwrite on an existing entry also bypasses the cap.

Volatility. Same as conquest: engine rebuilds respawn_params on STATE_Read. Two-phase restore via _swarmed_pending. 60s scanner re-applies via set_shared_spawn.

#### Infestation (exclusive spawn)

infest_smart(smart_id, faction, level_id) calls xsmart_spawn.set_exclusive_spawn(smart, "ap_infest", faction, spawn_num). Sets smart.faction_controlled to a non-nil value (activating the engine's faction gate at line 1667) and adds ONE respawn_params entry with a .faction field matching the infesting faction. LTX entries have no .faction so they fail the gate (nil == faction is false). Only the infest entry spawns. Exclusive replacement without deleting originals.

Faction re-apply. check_smart_faction runs every tick on online smarts and counts only IsStalker NPCs - monsters invisible. When only mutants occupy an online smart, self.faction reverts to default_faction, breaking the faction gate match. 60s periodic scanner re-applies smart.faction via set_exclusive_spawn for all infested smarts.

Per-level cap. can_infest_on_level(level_id) counts infested smarts with matching level_id. Rejects if count >= cfg.mutator_area_infest_max_per_level (default 3, MCM 1-5).

Volatility. Same as conquest: engine rebuilds faction_controlled, faction, respawn_params from LTX on every load. Two-phase restore re-applies all three in _on_game_load. 60s scanner handles ongoing faction reversion for online smarts.

Decay. Separate from conquest and swarm. cfg.mutator_area_infest_decay_hours (default 48). clear_exclusive_spawn reverts faction_controlled to nil, faction to default_faction, and removes the infest entry. Original LTX spawns resume on next try_respawn. Set to 0 to disable decay (permanent infestation).

Interaction with conquest and swarm. If a smart is infested AND conquered or swarmed, the exclusive spawn's faction gate suppresses the shared-spawn entries (they have no .faction field). Infest wins at runtime. All three data tables coexist independently; clearing infest restores the shared-spawn entries if they have not yet decayed.

---

## Causes

Per-cause inventory: see `ap_core_const.CAUSE` (canonical enum) and `ap_ext_cause_*.script` / `ap_ext_causes_*.script`. Cause generator contract: takes the callback args, returns either a payload tagged with the picked specific cause or nil. Generators self-gate the per-category rate (ap_core_limiter.check_cause_rate_limit + increment_cause_counter). Generators self-observe under the picked cause name. Generators are pure - no rate-limit awareness from outside, no counter manipulation, no side effects beyond the published payload.

### Cause classification

Four categories, each with its own mechanic for when and how the cause fires:

| Category | Mechanic | Cause | Consequence |
|----------|----------|-------|-------------|
| Reactions | engine event (death, pickup, healing) | builds payload from callback | perception scan from event position |
| Opportunities | squad tick + look-around | find_* + state-classify eligible siblings + RULES cascade | act using payload |
| Needs | squad tick + Hull score on stalker DTO | pick strongest drive | find satisfaction location + act |
| Instincts | squad tick + Hull score on mutant DTO | pick strongest drive | find satisfaction location + act |

Reactions are simple-mechanism (the engine callback IS the event). Opportunities, Needs, Instincts are radiant-mechanism (the squad tick is a heartbeat - only what the squad finds, scores, or picks IS the event).

### Where finds live: 1 -> N expansion

The find lives where 1 expands to N candidates.

- Reactive: 1 event -> N witnesses -> consequence find_squads (responder SCAN). Some reactive consequences also find_smart (destination SCAN) before the responder scan.
- Radiant: 1 squad -> 1 destination -> cause find_smart (destination SCAN). The SCAN lives in the cause generator; the consequence is action-only.

Reactive expands over witnesses (consequence-side). Radiant expands over the world but resolves the destination in the cause; the consequence consumes the resolved destination from the published payload.

### Theoretical foundations

Radiant causes split by what the threshold MEANS, not by how it is implemented. Both shapes use the same architectural pattern (DTO timestamp + threshold gate + arrival reset) but encode different theories.

Hull's drive reduction theory (1943). Behavior is driven by deprivation of a biological need. The longer the deprivation, the stronger the drive. Score formula (ap_ext_causes_needs.script:96-98): drive = weight * (elapsed / threshold)^2. Squared exponent makes overdue drives compete strongly against marginal ones. Used by NEEDS (9 stalker drives) and INSTINCTS (5 mutant drives). Threshold encodes how long the squad can tolerate the deprivation before the drive becomes urgent. Arrival action satisfies the drive and resets the timestamp.

Charnov's marginal value theorem (1976), optimal foraging theory. A forager exploits a patch, moves on, and the patch recovers before the next visit becomes worthwhile. Gate is binary: elapsed > threshold. No drive scoring; the squad either revisits the patch or doesn't. Used by stash (fill, loot, ambush) and area (conquer, swarm, infest). Threshold encodes patch handling time + travel time + recovery time between visits. Arrival action resets the timestamp.

DTO + timestamp + arrival-reset is the architectural shape both theories share. They differ in what the threshold means, not how it is implemented.

### Multi-answer drive (radiant generator pattern)

Each answer is a separate first-class cause paired 1:1 with its own consequence (sharing the noun per the radiant naming rule). "Multi-answer drive" is a code-structure quirk: a Hull-scored drive can be satisfied by any of several answers depending on squad identity (faction, species). The drive itself owns only the Hull threshold and the DTO timestamp field; everything else (enable, alignment, personality traits, filter) belongs to the individual answers.

The needs and instincts generators encode this as two tables:

| Table | Holds | Example |
|-------|-------|---------|
| NEEDS / INSTINCTS | one entry per drive: Hull weight, threshold cfg key, period gating, DTO field name | { instinct = "slumber", field = "last_slumber_at", weight = 2.5, dormant_period = true, threshold_key = "cause_slumber_threshold" } |
| CAUSES | one entry per specific cause (answer): cause const, short name, parent drive name, alignment subset, personality, filter | three slumber answers: slumber_field (cowardly, territory filter); slumber_lair (feral + predator, lair filter); slumber_surge (aberrant + predator, surge filter) |

Picker flow inside _on_smart:

1. Score every drive via Hull (_find_overdue_drives for instincts, _find_overdue_needs for needs).
2. Sort overdue drives descending by drive score.
3. Walk overdue drives top-down. For each drive, walk CAUSES entries with matching parent drive. Per cause: RULES (per-cause enable, alignment subset, personality roll), then SCAN (filter + find_smart_observed). First cause that publishes wins. Stop.
4. Each generator's `_on_smart` caps its own internal cascade at RADIANT_MAX_SCANS_PER_GENERATOR SCAN reaches; RULES rejections are free.

The DTO field is per-drive. When any of a drive's answers fires, the drive's timestamp resets (Hull drive reduction). Multiple answers compete to satisfy one drive.

cfg key layout:

- cause_<drive>_threshold: Hull threshold, one cfg key per drive (shared by every answer under that drive).
- cause_<answer>_enabled: per-cause enable, one cfg key per answer. For single-answer drives the answer name equals the drive name (feed, roam, pack, scatter), so the cfg key reads as cause_<drive>_enabled but is conceptually per-answer.
- Personality clamp is global, not per-cause. PERSONALITY_FLOOR (0.10) and PERSONALITY_CEILING (0.70) constants in ap_ext_const (line 174-175). Applies uniformly to every personality roll.

Used by ap_ext_causes_needs.script (9 drives, 16 answers) and ap_ext_causes_instincts.script (5 drives, 7 answers, multi-answer slumber). State-classifier generators (stash, area) use a different selection mechanism: the world peek (find_stashes / find_smart) classifies which sibling causes are eligible by world state (empty/non-empty, items_count, alignment) and emits an ordered list. The cascade walks the list and tests each sibling's RULES (MVT threshold + personality); first sibling to pass RULES reaches the 1 SCAN slot and publishes. No Hull scoring - eligibility comes from world state, not drive deprivation.

---

## Consequences

Per-consequence inventory: see `ap_core_const.CONSEQUENCE` / `ACTION` (canonical enums) and `ap_ext_consequences_*.script`. Reactive consequences carry their own RULES and SCAN. Radiant consequences are action-only - RULES and SCAN already happened in the cause generator.

Handler contract: takes the cause payload, returns a result code. Domain gates (alignment, species, personality) live in cause generators for radiant and in consequence handlers for reactive - always in ext, never in core. Dispatch order: shuffled per cause publish (reactive only).

### Consequence shapes

Two shapes by cause type.

Radiant: action-only. The cause generator delivered the squad and the destination smart in the payload. The handler routes the squad to the smart, registers the on-arrival action, calls ap_core_record.add_record to write the activity record (HUD marker + news compose), returns SUCCESS. No alignment, no species, no personality, no find - those happened in the cause generator.

Reactive RESPOND. Three phases: RULES (alignment, species, personality, payload validation) -> SCAN (find responder squads, optionally a destination smart) -> ACTION (per responder, route and record). At least one responder routed = SUCCESS. Examples: massacre_investigate, massacre_scavenge, basekill_support, basekill_flee, wounded_hunt, wounded_help, harvest_rob, harvest_haunt, squadkill_revenge, alphakill_targeted.

Reactive TRANSFORM. Two phases: RULES (payload validation, threshold or cap checks) -> ACTION (mutate state, register, dispatch on the entity in the cause payload). No responder loop. Examples: alpha_promote.

Gate order within the RULES phase (where present): alignment -> species -> personality -> match -> validation.

### Chase pattern

Three reactive consequences (squadkill_revenge, harvest_rob, alphakill_targeted) pursue a non-stationary target. Two target kinds share one lifecycle:

- Player target: `move_actor_chasers` -> `script_actor_target` sets `scripted_target = "actor"`, which the engine re-resolves to id 0 every update, so pursuit is continuous and the actor never "arrives".
- Squad target: `move_squad_chasers` -> `script_squad_target` drives continuous coordinate homing. A squad id is not a natively-resolvable scripted target (vanilla `get_script_target` / `get_current_task` resolve only `"actor"` and smart names), so `ap_core_chase` installs two guarded `sim_squad_scripted` monkeypatches at on_game_start, both keyed on the broker `_ap_scripted_squads` registry (`entry.target_squad_id`):
  - `get_current_task` builds a `CALifeSmartTerrainTask` from the target's live vertices each call. This is the only movement input the offline group brain consumes (`alife_online_offline_group_brain.cpp:80-84`), and it THROW2s on a nil task - the hook never returns nil (it returns a constructed task, or the captured original, which itself falls back to `get_alife_task`).
  - `get_script_target` returns the target id so the online reach-task squad fallback fires (`xr_reach_task.script:251-254`) and `specific_update` keeps re-pinning `assigned_target_id`. Replacing the whole method also sidesteps the clean-vanilla numeric-id `obj:clsid()` crash (`sim_squad_scripted.script:139`).

Both hooks delegate to the captured original for any squad without a chase entry, so they chain safely with warfare's identical `get_current_task` hook regardless of load order. The chaser homes until the target dies (`am_i_reached` is `npc_count()==0`); a combat death releases the chase. The smart-hop relay (`make_move_smart_chasers` / recursive `make_on_arrive`, which trailed a moving target by one smart-to-smart leg) is deleted.

#### Leash

`ap_core_util.evaluate_chase(squad, entry)` is the single retention decision for both chase kinds, returning `(verdict, reason)` with verdict in `VERDICT.{KEEP, PAUSE, RELEASE}`. It composes small offline-safe `is_*` predicates, RELEASE-before-PAUSE and cheap-before-expensive, short-circuiting on the first hit - the same `(bool, reason)` shape as the broker off-map despawn chain. Pure (no side effects), so the periodic scan and the dispatch both call it.

- RELEASE: `_is_target_lost` (squad target gone or wiped; N/A for the actor), `_is_no_longer_enemy` (opt-in via `entry.check_relation` - faction chases set it, mutant species chases omit it because `is_factions_enemies` is false for undefined mutant pairs), `_is_target_story_protected` (target carries a story id), `_is_target_off_map` (chaser and target on different maps - stateless, so a chase never follows across a level change or leaks into the off-map session system).
- PAUSE (resume when the condition clears): `_is_target_in_base` (target sheltered in a base smart), `xlevel.is_surge`. Every chaser pauses uniformly; no faction exemption.

The scan (`ap_core_broker._update_chase`) acts on the verdict. RELEASE unscripts. PAUSE stops driving the engine target - the squad-target hook returns nil while paused, and the actor target is released and re-set on resume - with a min-hold (`CHASE_MIN_PAUSE_SEC`, hysteresis against base flicker) and a max-pause cap (`CHASE_MAX_PAUSE_SEC` -> release). KEEP resumes a paused chase and reasserts the actor target. The dispatch (`script_squad_target`) runs the same `evaluate_chase` on the proposed entry stub and refuses any start that is not KEEP. Both chase kinds also share the one ledger TTL (`SCRIPTED_SQUAD_TTL`), checked above the chase/smart split in the scan.

Persistence is the seed only: `_ap_scripted_squads` stores `target_squad_id`; the hooks regenerate the task each tick, so the engine saves neither the task nor `assigned_target_id`. A chase survives save/load and resumes within one scan.

### Domain gates

Three gates filter who participates: alignment (hard, deterministic), species (hard, deterministic), personality (probability, non-deterministic). All three live in ext, never in core.

- Alignment: can this faction do this consequence at all. Static set lookup.
- Species: filters mutant kinds (cowardly, feral, predator, aberrant). Stalkers have no species and pass automatically. Resolved once per squad, cached.
- Personality: rolls the probability of acting given relevant trait scores. avg(traits), clamped to [PERSONALITY_FLOOR, PERSONALITY_CEILING]. Inverted traits (only INV_AGGRESSION, INV_DISCIPLINE, INV_TERRITORY are defined) resolve as 1 - base before averaging.

Not every consequence uses all three. Human-only consequences skip species. Some have no gates beyond the enable toggle. When gates are present, order is alignment -> species -> personality.

Where they apply by cause type:

- Radiant: gates run in the cause generator on the ticking squad. Squad is both sensor and responder.
- Reactive same-faction: gates run in the consequence handler on the event faction (victim, wounded). Responders inherit that faction.
- Reactive cross-faction: alignment set is passed as the responder filter to find_squads. Species and personality run per-responder inside the find loop.

### Alignment

Hard filter: can this faction do this consequence at all. Static hash sets in ap_ext_const, O(1) per check. Zombied is excluded from alignment_human and therefore from all human consequences globally.

Human factions follow GSC's moral axis (Ai.doc:65-76, тип характера):

| Table | Factions | GSC origin |
|-------|----------|------------|
| alignment_principled | dolg, army, monolith, isg | Principled: follows rules, organized |
| alignment_selfserving | stalker, csky, ecolog, freedom, killer | Self-serving: independent, own goals |
| alignment_unprincipled | stalker, freedom, killer, csky | Chaotic neutral: own rules, not criminal |
| alignment_outlaw | bandit, renegade, greh | Chaotic evil: criminals |
| alignment_human | all 12, no zombied | Union of principled + selfserving + outlaw |
| alignment_naturalist | stalker, csky, freedom, ecolog | AP subset: zone dwellers |

Mutant species alignments follow GSC's creature groups (monstry.doc:4). Keyed by species string (xcreature.get_mutant_species), not engine faction:

| Table | Species | GSC origin |
|-------|---------|------------|
| alignment_mutant | all 7 monster factions | Fast gate for find_squads (engine player_id) |
| alignment_mutant_cowardly | flesh, zombie, tushkano, rat, karlik | Timid, flees danger, bottom of food chain |
| alignment_mutant_feral | dog, pseudodog, boar, snork, cat, gigant | Pack / herd, reactive aggression, brute apex |
| alignment_mutant_predator | lurker, bloodsucker, psysucker, chimera, fracture | Solitary hunters, ambush, pursue wounded |
| alignment_mutant_aberrant | controller, burer, poltergeist, psy_dog | Psychic, lair-bound, supernatural |

Keying trap (read this before touching mutant alignment lookups). Two namespaces:

- alignment_mutant: keyed by engine faction (squad.player_id, e.g. "monster_predatory_day"). Use [squad.player_id].
- alignment_mutant_cowardly / _feral / _predator / _aberrant / _night / _day and any merge of them (e.g. _alignment_conquer_mutant, _alignment_lair, _alignment_pack): keyed by species string (xcreature.get_mutant_species result, e.g. "bloodsucker"). Use [species].

The two namespaces have zero overlap. alignment_mutant_predator[squad.player_id] always returns nil - silent dead branch that looks correct because no error is raised. Check the table comment in ap_ext_const.script and the variable name in the call site (squad.player_id vs species). If a derived / merged table does not have an explicit comment, trace its inputs.

Activity alignment axis (independent of behavioral axis, gates day / night cycle):

| Table | Species | GSC origin |
|-------|---------|------------|
| alignment_mutant_night | bloodsucker, psysucker, lurker, chimera, zombie, fracture | monstry.doc: bloodsucker noch'yu vykhodit. Engine: monster_predatory_night, monster_zombied_night. |
| alignment_mutant_day | all other species | Engine: monster_predatory_day, monster_zombied_day. Default for lair-bound species. |

Consequences compose tables with xtable.merge (set union) and xtable.subtract (set difference) at module load.

### Personality

Probability layer: how likely is an eligible faction / species to act. Runs only after alignment passes. All checks happen in ext consequence code via ap_ext_util.check_personality, never in core.

Stalker factions have 7 traits: aggression, greed, survival, perception, territory, relation, discipline. Mutant species have 5: aggression, survival, territory, perception, relation. Each consequence declares at most 2 relevant traits. The check averages those traits for the faction / species and rolls math.random() against the result, clamped to [PERSONALITY_FLOOR, PERSONALITY_CEILING].

Formula: chance = clamp(avg(relevant_traits), PERSONALITY_FLOOR, PERSONALITY_CEILING). PERSONALITY_FLOOR = 0.10, PERSONALITY_CEILING = 0.70 (ap_ext_const.script:174-175). No per-consequence weight. Floor ensures even unfavorable factions act occasionally; ceiling ensures even favorable factions fail sometimes.

Inverted traits: traits prefixed with INV_ resolve as 1 - base_value before averaging. Used for behaviors driven by absence of a quality (fleeing gated by INV_DISCIPLINE + INV_TERRITORY: low discipline and low territorial attachment = more likely to flee). Only INV_AGGRESSION, INV_DISCIPLINE, INV_TERRITORY are defined (ap_ext_const.script:167-169).

Trait value design: tiered values in 0.10 steps (0.10, 0.20, ..., 0.90) to ensure clean math. Survival is a flat band (0.40-0.60) for all factions and species. Biological needs are universal drives, not faction differentiators. Each value is a direct probability grounded in GSC lore (monstry.doc, Ai.doc).

### Range tiers

Every consequence searches within a range that matches the squad's awareness. Two tiers, grounded in GSC's PersonalEyeRange (EFC design docs, circa 2002) and validated against empirical smart terrain spacing (14 levels measured via TestZone, see doc/library/modding/level-geometry.md).

| Tier | Constant | Distance | Who |
|------|----------|----------|-----|
| EyeRange | RANGE_EYE | 200m | all (line of sight) |
| RadioRange | RANGE_RADIO | 500m | stalkers (PDA / radio) |
| ScentRange | RANGE_SCENT | 500m | mutants (scent tracking) |

EyeRange covers the p90 nearest-neighbor distance on 13 of 14 measured levels. A squad at any smart can see 1-3 neighboring smarts and several stashes within 200m. RadioRange and ScentRange cover the full operational radius. Same distance today (500m), independently tunable.

Each cause type maps to a range based on how the squad learns about the opportunity:

| Cause type | Stalker range | Mutant range | Rationale |
|-----------|---------------|--------------|-----------|
| Opportunities (stash, territory) | EyeRange | EyeRange | Squad sees what is nearby on arrival |
| Reactions (kills, massacres, wounded) | RadioRange | ScentRange | Stalkers hear over radio, mutants smell blood |
| Needs (hunger, sleep, shelter) | RadioRange | - | Squad knows campfire / trader locations from PDA |
| Instincts (feed, sleep, explore) | - | ScentRange | Mutants track by scent across the area |

Three tiers create a natural separation: opportunities are local and opportunistic (200m), stalker coordination reaches further via radio (500m), mutant hunting extends through scent (500m). You act on what you see, respond to what you hear, hunt what you smell.

### Day / night cycle

All radiant causes are gated by the active / dormant period system. Reactions are not. Drives (needs + instincts) declare a per-drive flag (active_period = true or dormant_period = true). Opportunities (stash, area) gate on active_period implicitly - every opportunity fires only when the squad's identity is in its active period.

ap_ext_util.is_active_period(identity) resolves the current period for any species or community. Nocturnal species (alignment_mutant_night) are active at night (20:00-05:00). All others (stalkers, diurnal mutants) are active during day (05:00-20:00).

Stalker needs:

| Drive | Flag | Effect |
|-------|------|--------|
| hunger | active_period | day only |
| sleep | dormant_period | night only |
| rest | dormant_period | night only |
| heal | active_period | day only |
| shelter | dormant_period | night only |
| supply | active_period | day only |
| money | active_period | day only |
| job | active_period | day only |
| social | active_period | day only |

Mutant instincts:

| Drive | Flag | Effect |
|-------|------|--------|
| scatter | (none) | any time (transitional, moving to FLEE family per n102) |
| feed | active_period | active period only |
| slumber | dormant_period | dormant period only |
| roam | active_period | active period only |
| pack | active_period | active period only |

Opportunities (no per-cause flag, all implicit active_period):

| Cause | Effect |
|-------|--------|
| stash_loot / stash_ambush / stash_fill | stalkers active period (day) |
| area_conquer | stalkers active period (day) |
| area_swarm | mutant species active period (day for diurnal, night for nocturnal) |
| area_infest | mutant species active period (day for diurnal, night for nocturnal) |

---

## Rules

Reference vocabulary first, then the rule set split into universal, radiant, and reactive.

### Reference vocabulary

Two cause types: REACTIVE (triggered by an engine event, fans out 1:N to consequences) and RADIANT (triggered by the squad tick, 1:1 with its consequence).

Two consequence file shapes: _set (hand-written N handlers, used for per-handler quirks) and CONFIGS factory (one CONFIGS table + one closure builder generates N handlers from a single body).

Five result codes: SUCCESS, FAILED_RULES, FAILED_SCAN, FAILED_ACTION, DISABLED. The handler (`entry.handler` in `ap_core_consumer`) returns one of the first four when it runs. DISABLED is a rules-layer skip emitted by the consumer pre-gate when a consequence's MCM toggle is off; the handler is not called.

### Universal rules

1. Core never imports ext. All domain logic reaches core through registered function references. Ext provides only behavior (predicate bodies, handler bodies, domain data); ext does not know about gates, events, or flow control. Flow (gate chains, evaluation policy, dispatch) is core.
2. m_data accepts only primitives and tables of primitives. Functions, userdata, metatables are silently dropped on save.
3. Every consequence returns one of SUCCESS, FAILED_RULES, FAILED_SCAN, FAILED_ACTION. Missing or malformed return = error.
4. Published causes are always specific names (cause:hunger_campfire, cause:massacre). Umbrella names (NEEDS, REACTIONS) are categories, never published.
5. Every cause has its own MCM toggle. No file-level master toggle.
6. Every consequence has its own MCM toggle.
7. News entries carry the published cause and consequence verbatim. Never an umbrella constant.
8. Every cause registration declares a CAUSE_CATEGORY (REACTIONS, NEEDS, INSTINCTS, OPPORTUNITIES). Category drives rate-limit grouping; it is never published.
9. Domain gates (alignment, species, personality) live in ext, never in core. Location depends on cause type - see radiant and reactive rules.
10. Runtime smart terrain mutations are rebuilt from LTX on load. Two-phase restore re-applies them after entities exist.
11. scripted_target overrides SIMBOARD's target_precondition. AP consequences enforce faction safety inline; runtime job availability is checked at arrival and re-checked during the gulag hold via xsmart.has_jobs_for.
12. Many cause attempts fail by design - RULES (especially personality) make outcomes likely or unlikely. Variance comes from cascade ordering, not from clamping personality.
13. One generator per family. Bundle when causes share input or scoring (Hull cascade over drives, state-classifier over a peek). Use separate generator files when causes have independent triggers or scans (every reactive cause has its own file).
14. Cause publish contract: a generator must not publish if no consequence can act on the event. Causes serving a subset of alignments must filter at the cause level.

### Radiant rules

1. The actor is always the ticking squad. No actor scan in radiant.
2. Cause and consequence are 1:1 and share the noun. Multi-answer drives split into multiple causes; each cause has its own 1:1 consequence.
3. The cause generator owns RULES (toggle, alignment, personality, threshold) and SCAN (find_smart). The consequence is action-only - resolve squad and smart from the payload, route the squad, record the event, return SUCCESS.
4. The generator cascades internally (cause-to-cause within the generator, top-down) and externally (generator-to-generator at the producer). Hull-scored generators (needs, instincts) order drives by Hull score and walk CAUSES under each overdue drive. State-classifier generators (stash, area) order siblings by world-state eligibility and walk them in that order. Both then cascade on RULES, first publish wins.
5. No fallbacks inside a single cause. One SCAN with one filter. Variant outcomes are separate causes.
6. Per-generator SCAN budget: RADIANT_MAX_SCANS_PER_GENERATOR = 1. A slot is consumed only when a cause passes RULES and reaches its world SCAN (find_smart / find_squads). RULES rejections (toggle, alignment, personality, threshold, period) are free and do not count. At most one SCAN reach per `_on_smart` call; any subsequent cause that reaches SCAN returns BUDGET_EXHAUSTED. Budget is local to one `_on_smart` invocation and resets on the next call. Internal tuning, not MCM.
7. Stalker and mutant get separate causes for the same world state - needs (stalker) and instincts (mutant) are distinct families with distinct DTOs.
8. CONFIGS factory is the default consequence file shape. _set is vestigial for radiant.
9. on_arrive: the DTO reset is unconditional (online or offline - the abstract state advances either way). World mutation (item consume, trade, inventory operations) runs only when online.

### Reactive rules

1. A reactive cause subscribes to one engine callback or one callback-family constant. Callback families collapse multiple engine paths into one logical cause: HARVEST_CALLBACKS = { ACTOR_ON_ITEM_TAKE, NPC_ON_ITEM_TAKE }, WOUNDED_CALLBACKS = { AP_NPC_MEDKIT_USE, ACTOR_ON_ITEM_USE }.
2. Cause:consequence is 1:N. Multiple consequences fan out independently per publish.
3. RULES split across three locations:
   - Cause-level event RULES: universal event criteria (does this event qualify for publishing at all). Runs once per event. Fail = no publish.
   - Consequence-level event RULES: per-consequence event criteria (does THIS consequence apply to this event). Runs once per consequence dispatch. Fail = skip this consequence.
   - Per-responder RULES: per-responder identity filter (personality, species). Runs per responder during the cascade.
4. Consequence shapes: RESPOND (find responder squads, dispatch each) or TRANSFORM (act directly on the cause-target entity, no responder loop). RESPOND consequences may run two SCANs - destination SCAN (find_smart) and responder SCAN (find_squads) - typically destination first since responder SCAN may use destination to exclude already-arrived squads.
5. Responder SCAN returns up to a per-consequence configurable max_count (default 2).
6. Stalker and mutant get separate consequences subscribed to the same cause. massacre fires both massacre_investigate (stalkers) and massacre_scavenge (mutants) independently.
7. Per-publish consequence cap: REACTIVE_MAX_CONSEQUENCES_PER_CAUSE = 2. One unit = one consequence run, independent of how many responders the consequence cascaded through.
8. _set is the natural file shape for hand-written quirks (varying SCAN filters, varying actions, varying per-responder logic). CONFIGS factory only when consequence bodies are uniform.

---

## Integration API

AlifePlus exposes three levels of integration for external mods. See integration.md for full examples and code templates.

![Integration Levels](img/integration-levels.png)

### Level 1: Listen (xbus subscriber)

Subscribe to cause events via xbus. No AP code dependency - xlibs only.

```lua
xbus.subscribe("cause:massacre", function(data)
    -- data.position, data.level_id, data.squad_id
end, "my_mod")
```

The heartbeat itself is public: ap_core_callbacks fires `ap_squad_on_change` through the vanilla callback registry for every squad in the Zone (including squads AP refuses to touch), so a co-installed mod subscribes natively. The event exists only when AlifePlus is installed, since AlifePlus declares and fires it:

```lua
RegisterScriptCallback("ap_squad_on_change", function(squad, changed, prev, curr)
    -- changed: array of field names ("gvid", "online", "smart_id", "action"),
    -- nil = round-robin turn with no transition
    -- prev/curr: last and current 4-field snapshots; read-only, do not retain references
end)
```

### Level 2: Register (pipeline participant)

Register a cause generator with ap_core_producer.register(config, generator) and a consequence handler with ap_core_consumer.register(name, config, handler). The framework handles gates, protection, rate limiting, tracing, arrival, and cleanup.

| Parameter | Producer | Consumer |
|-----------|----------|----------|
| name | (handler ref dedupes) | Consequence identifier (also used as trace key, rate limit key, arrival key) |
| config.callback | Engine callback name | - |
| config.cause_type | RADIANT or REACTIVE | - |
| config.category | CAUSE_CATEGORY enum (REACTIONS / NEEDS / INSTINCTS / OPPORTUNITIES). Required. Drives per-category rate-limit grouping | - |
| config.event | - | Cause event to subscribe to |
| config.condition | - | Optional pre-gate (typically MCM enabled flag) |
| config.on_arrive | - | Optional arrival handler function |
| handler | Predicate function (self-gates per-category rate) | Consequence handler function |

### Level 3: Coordinate (external mod integration)

Register a squad ownership filter to prevent AP from scripting squads your mod controls.

```lua
ap_core_broker.register_owner("my_mod", function(squad)
    return squad.my_mod_flag == true
end)
```

Squads matching any registered ownership filter are excluded from AP at the protection gate (producer), in find_squads results (consequence), and via get_owner queries. Gated by MCM allow_external_ownership. Warfare is registered by default in ap_core_compat.

Activity record queries (Coordinate level): foreign integrators that need to know what AP is currently doing on a squad call ap_api.get_record({squad_id, assigned=true}) for the live entry, or ap_api.get_records(opts) for a bulk filter. Each entry carries denormalized display facts (subject_*, other_*) so callers do not re-resolve faction or species. Integrators that need the live scripted-squad set call ap_api.get_scripted_squads().
