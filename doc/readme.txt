AlifePlus: Emergent A-Life for STALKER Anomaly, by Damian
- Version: 1.7.6 (xlibs 1.7.6, demonized 20260601)
- Manifesto: https://github.com/damiansirbu-stalker/AlifePlus/blob/main/doc/manifesto.md
- Integration guide: https://github.com/damiansirbu-stalker/AlifePlus/blob/main/doc/integration.md
- Changelog: https://github.com/damiansirbu-stalker/AlifePlus/blob/main/doc/changelog
- Russian / На русском: https://github.com/damiansirbu-stalker/AlifePlus/blob/main/doc/readme_ru.txt
- Bugs, suggestions: https://github.com/damiansirbu-stalker/AlifePlus/issues

! Please use the RESET button in MCM when updating to a new version !

You are not special.

AlifePlus is a reactive A-Life framework for STALKER Anomaly.
It extends A-Life with event-driven emergent behavior, built on GSC's original AI design.
It intercepts engine events, classifies them into causes, and dispatches consequences that extend the simulation.
Behaviors chain. One consequence becomes the cause of the next.
Squads investigate massacres, hunt artefact carriers, claim empty territory, and act on hunger, sleep, and social needs.
Everything that happens to the player happens to NPCs and mutants alike.
Every action traces back to a cause, a world state, and a mechanic.

AlifePlus draws on Roadside Picnic and the original STALKER vision.
The Zone runs on its own rules, and the actor is just another entity inside it.
Each faction acts on its identity: Duty holds ground, Bandits ambush and loot, Loners chase artefacts, Ecologists research, Military stays close to base, Monolith never retreats, Renegades scatter at first contact.
Alignment determines what a faction can do at all, and personality determines how likely it is to act.

Every action has a systemic cause. The simulation runs whether you are there or not.
Structural invariants make this possible:
- Event-driven contract: nothing runs unless the engine says something happened.
- Physical simulation guarantee: consequences use entities already in the simulation, never spawned, teleported, or fabricated.
- Item transfer only: items move between carriers (NPCs, stashes, traders). AlifePlus never invents new section names. Every materialized item is a section the engine already knows from vanilla configs.
- Money flows match vanilla: NPCs gain money when they sell items at supply traders, spend money when they buy ammo and consumables. No money moves through stash loot, stash fill, or any other AP flow.

The economy follows the same logic. Real items move between real stalkers, and real needs drive their decisions.

What you'll notice:

Reactive responses:
- Squads reroute after nearby combat deaths.
- Scavengers and investigators converge on massacre sites.
- Allies retaliate after a squad wipe.
- Predators close in on wounded targets.
- Same-faction allies rush to help the wounded.
- Outlaws hunt whoever picked up an artefact.

Active opportunities:
- Squads route only to smart terrains with real animation jobs (campfires, shelters, traders, patrols). Idle parking is filtered out.
- AlifePlus fills the smart terrains Anomaly ships with NPCs, so every animpoint, patrol path, and smartcover the game ever wrote finally gets used.
- NPCs walk to stashes to loot, ambush, or fill them with supplies.

Territory and population:
- Stalkers and mutants claim empty outposts.
- Conquests decay if nobody holds them.
- Mutants infest lairs and buildings as nests.
- Radiant dispatches weigh tactical soundness. A squad that would leave its base undefended by its faction, or pile onto a destination another scripted squad already targets, scores low and usually stays.

Alpha mutants:
- Alpha mutants emerge from combat and accumulate kills.
- Killing an alpha draws same-species hunters.

Trade and needs:
- Stalkers visit supply traders to sell surplus and restock per a rank-tiered policy. Rookies carry basic ammo, medkits, and bandages. Veterans add premium ammo (AP, hollow-point, flechette), grenades, and larger consumable bands.
- Squads loot a stash by per-category bands (ammo matching equipped pistols and rifles, medkits, bandages, food, drink, antirad, grenades). Quest items skip the whole stash. An MCM toggle skips stashes containing crafting items. The stash is cleared at the end of the loot pass.
- Squads fill stashes with squad surplus above each category's max, capped per event. Quest items and equipped gear are never deposited.
- Hunger, fatigue, heal, social, and outpost needs drive campfire and base behavior. Arrival satisfies the need. Inventory items are not destroyed.

Day/night cycle:
- Stalkers work and trade by day.
- Stalkers rest and sleep at campfires by night.
- Nocturnal predators hunt at night and retreat to lairs at dawn.
- Diurnal mutants follow the opposite cycle.
- Prey scatters when a predator moves through.
- Prey returns once the predator dies.

---

Example scenario (systemic interaction):

- A stalker on Cordon at dusk spots a stash by the road ("stash" cause). His faction's greed is high enough to take it ("loot" consequence).
- A bandit squad saw it first. They ambush him at the cache ("ambush" consequence).
- The player, heading to the same stash, watches the fight and wipes the bandits as the stalker bleeds out.
- The bodies pull cowardly mutants in from the brush ("massacre" cause, "scavenge" consequence).
- The dead stalker's faction sends a squad to find out what happened ("investigate" consequence).
- A nearby bandit crew catches the wipe call and turns toward the player ("squadkill" cause, "revenge" consequence).
- Loners on a patrol route cross the area and shoot the scavenging mutants ("job" consequence).
- The patrol is tired now ("needs" cause). They head for a campfire to smoke and tell the story ("social" consequence).
- None of it was scripted.
- The player walked into something already in motion.

Example scenario (escalation chain):

- An artefact spawns in an anomaly field. A loner needs money and heads there to pick it up ("needs" cause, "money" consequence).
- He grabs the artefact and turns for the trader ("harvest" cause).
- Outlaws catch the channel chatter and converge to rob him on the return ("rob" consequence).
- A poltergeist senses the artefact and haunts the pickup site ("haunt" consequence).
- The road back is already loud. Dogs and a flesh pack move on the noise. A chimera trailing them takes kills in the firefight and turns ("alpha promote" consequence).
- The player kills the alpha. Another chimera on the same tier reads the kill and turns to pursue ("alphakill targeted" consequence).
- A friendly stalker reports the alpha-down hours later as PDA news.
- The surviving chimera with its pack takes a nearby smart terrain as a nest ("infest" consequence). The infestation overwrites the original spawns.
- A bandit squad sees the ground empty around the lair and takes a neighboring smart ("area_conquer" consequence). Their spawns join the originals as a shared injection.
- The cycle continues. The Zone does not care.

Example scenario (economy loop):

- A loner squad finishes a hunt at an anomaly field ("needs" cause, "money" consequence). They have artefacts, mutant parts, and a spare PMM.
- They are hungry now ("needs" cause). They stop at the nearest campfire to rest by the fire ("hunger" consequence).
- They head for Sidorovich at Cordon to sell ("supply" consequence).
- Sidorovich buys the artefacts, mutant parts, the spare PMM, and surplus at engine prices.
- The loner restocks ammo for his AK74 and tops up his medkits and bandages, per his rank-tiered policy.
- The squad has rubles and stocked ammo now. They walk back to the anomaly field for the next hunt.
- The same loop the player walks, running for everyone.

---

What AlifePlus avoids:

A-Life mods often scan the world every frame (O(n) per tick), hijack squads by overwriting engine variables, swallow crashes silently, and accumulate stale state across saves.
The result is poor performance, entity leaks, ghosting, save corruption, and mod conflicts that get worse the longer you play.
AlifePlus avoids these by construction. The framework subscribes to engine callbacks instead of polling, squads stay engine-owned, every call is timed and traced, and every bookkeeping entry carries a TTL.

---

Decision system: nothing is purely random.

Every consequence runs through range search, alignment, personality, and world state before it commits. Range search narrows the candidate pool. Alignment decides whether the action is allowed at all. Personality decides probability. World state confirms the action makes sense right now. A consequence fires only when all of them agree.

Range search:

  AlifePlus searches within the squad's awareness, not across the whole map.
  - EyeRange: what the squad can see, like stashes and unclaimed territory.
  - RadioRange: what stalkers hear over the PDA, like massacres, base attacks, and trader locations.
  - ScentRange: what mutants smell, like corpses, wounded prey, and blood trails.

  Range tiers are grounded in GSC's PersonalEyeRange (EFC design docs) and validated against in-engine smart-terrain spacing across the Anomaly level set.

Alignment (hard gate):

  Faction identity decides whether an action is even possible.

  Stalker factions follow GSC's principled / self-serving / unprincipled / outlaw axis.
  The bars are structural and cannot be tuned: military squads cannot flee, ecologists cannot conquer, renegades cannot investigate, and outlaws cannot help the wounded.

  Mutant species sit on independent behavioral and activity axes.
  Behavioral tiers: cowardly (flesh, zombie, rats), feral (dogs, boars, snork, gigant), predator (bloodsucker, chimera, lurker), and aberrant (controller, burer, poltergeist).
  Activity axis is binary: nocturnal (bloodsucker, lurker, chimera, zombie, fracture) versus diurnal.
  The tiers form a food chain in which lower tiers scatter from higher tiers, with aberrant species at the top.

Personality (probability layer):

  Per-faction and per-species trait profiles, with values derived from GSC's AI design documents.
  Stalker profiles carry aggression, greed, survival, perception, territory, relation, and discipline. Mutant species carry the same set minus greed and discipline.
  Some traits (aggression, territory, discipline) have inverted variants for consequences that fire on low scores. A flee consequence reads inverted aggression so a low-aggression faction is the one most likely to flee.
  Each consequence binds to its relevant traits and averages them.
  The averaged value is clamped to a floor and ceiling tunable in MCM, and used as the probability of acting.

World state (scan layer):

  The scan phase queries live world state before the consequence acts.
  Smart terrain capacity, ownership, current engine-native squad control, recent activity, faction territory bookkeeping, time of day, online/offline status.
  Stale or contradictory state rejects the consequence. A squad already scripted by another mod is left alone. A full smart terrain rejects new arrivals. An emission or psi-storm suppresses the news queue.

---

Features:

Causes and consequences aggregate into hundreds of combinations.

Reactions

  World events trigger responses.

  Massacre
    - Scavenge - Cowardly mutants scavenge corpses at the massacre site.
    - Investigate - The victims' faction sends nearby squads to investigate.
    - Loot - Outlaws (Bandit, Renegade, Greh) strip the corpses for weapons and gear.

  SquadKill
    - Revenge - Same-faction squads pursue the killer as it moves.

  BaseKill
    - Support - Nearby friendly squads rush to reinforce the base under attack.
    - Flee - Squads at the base evacuate to the nearest friendly position.

  Alpha
    - Promote - Mutants accumulate kills and level up. Promotion is the gate for AlphaKill hunters.

  AlphaKill
    - Targeted - Same-species mutants on the same level hunt the killer.

  Wounded
    - Hunt - Predator and aberrant mutants sense weakness and close in.
    - Help - Nearby same-faction squads rush to help the wounded.

  Harvest
    - Rob - Nearby outlaws pursue whoever picked up the artefact.
    - Haunt - Aberrant mutants converge on the artefact pickup site.

Opportunities

  A squad sees what is nearby and acts on it.
  A per-squad drive cooldown prevents repeated firing.

  Stash
    - Loot - A stalker squad spots the stash, walks there, and loots it.
    - Ambush - An outlaw squad spots the stash and camps it as bait.
    - Fill - A stalker squad spots the stash and hides supplies inside.

  Area
    - Conquer - Stalker factions claim empty territory (Ecologists excluded). Mutants use Swarm.
      Conquest adds the conqueror's spawns alongside the originals as a shared injection (both streams stay active).
      Territory decays over time (MCM configurable).
    - Swarm - Feral, predator, and aberrant mutant squads claim empty smart terrains.
      Swarming adds the swarming species' spawns alongside the originals as a shared injection (both streams stay active).
      Territory decays over time.
      Decay, cap, and spawn count tune independently from Conquer in MCM.
    - Infest - Feral, predator, and aberrant mutants claim smart terrains as nests.
      Only squads carrying an alpha can infest.
      Infestation replaces original spawns with the infesting species as an exclusive injection (originals are suppressed).
      A per-level cap limits spread.

Needs

  Stalkers have human needs.
  Drives inspired by Maslow-Hull are scored by how long since each was last fulfilled.
  The most deprived need wins.
  - Hunger - The stalker walks to a campfire to eat by the fire.
  - Sleep - The stalker walks to a campfire during dormant hours to sleep.
  - Rest - The stalker walks to a campfire to rest.
  - Heal - The stalker walks to a shelter when injured.
  - Shelter - The stalker walks to a safe location when exposed too long.
  - Money - The stalker searches anomaly fields for artefacts or hunts mutant lairs.
  - Supply - The stalker walks to a trader, medic, or mechanic (anyone wired to dm_init_trader in vanilla) to sell surplus and restock per a rank-tiered policy. Veterans add premium ammo, grenades, and larger consumable bands. Rookies carry basic ammo and standard medkits.
  - Job - The stalker guards outposts and checkpoints, explores the Zone, or researches anomalies.
  - Social - The stalker walks to a campfire or base for company.

  Cross-map sibling rows: Social, Supply Trader, and Job Explore each ship a paired off-map row. Peaceful sociable factions (Ecologists, Clear Sky, Freedom, Loners) visit other camps for company. Peaceful greedy factions cross to another map to restock when no on-map trader is reachable. Peaceful curious factions (Ecologists, Clear Sky) scout neighbor maps. Army, Monolith, and Zombified squads never travel off-map.
  Reach grows with player progression. Stalkers reach the next map once Yantar X-16 is shut down, the next after the Brain Scorcher is deactivated at Radar, and master-rank squad commanders push one map further. Each step is tunable under MCM > World > Off-map.
  Squads return home after a few in-zone days. Travellers stuck off-map are released after a week to keep level populations clean. Dispatches are also capped per source map within a sliding two-day window so no level bleeds traffic forever.

  Arrival satisfies the need. Inventory items are not destroyed by the arrival itself. Engine-side combat behavior (medkit, bandage, stim usage in firefights) is independent and unchanged.

Trade

  AlifePlus runs a category-based NPC buy/sell cycle on arrival at trader smarts. NPCs visit a trader, sell surplus from their inventory, and restock ammo, grenades, and consumables per a per-rank policy. All 20 vanilla trader smarts are covered, from Sidorovich at Cordon to the Monolith trader in Pripyat. Built on Alundaio's buy/sell core (2013, maintained by Tronex through 2019), modernized to accept extension instead of dirtying item files, and to carry depth the original row schema could not express.

  Categories instead of items. Alundaio's row schema lived inside each tradeable item's LTX: every section had to declare its own buy_sell row, and modpacks adding items had to dirty the item files to participate in trade. AlifePlus declares categories in ap_trade_policy.ltx (DLTX-overridable). xinventory resolves any section to its category through the engine's own item buckets. A Boomsticks AP round, a GAMMA addon medkit, an EFP grenade all classify automatically through their existing kind and class fields. The modder adds the item, the mod ships, AlifePlus trades it. No per-modpack LTX maintenance, no item file edits.

  Per-rank policy. Two blocks split policy by character rank: rookie (rank below 12000) and veteran (rank 12000 and above). Each block lists categories with a min, max band: medkit, bandage, antirad, stim, pill, grenade, food, drink, plus per-slot ammo tiers (ammo_slot_2_t1/t2 pistol basic/premium, ammo_slot_3_t1/t2 rifle basic/premium). Entry order sets the buy priority.

  Real equipped weapons drive ammo picks. The pistol's ammo class is read at runtime, sorted by k_ap, split into basic and premium tiers. The rifle's ammo class splits the same way. A veteran with an SVD gets basic and premium 7.62x54R. A rookie with a PM gets basic 9x18 only. Within a tier, the cheapest section is picked first so NPCs with limited cash actually fill their slot. Consumables walk cheapest-first from the category's section set.

  Sell phase. The stalker walks his inventory and drops anything whose category count exceeds the policy max. Equipped items (whatever sits in his slots, knife to backpack), quest items, anim items, and money are never touched. Three runtime untouchables also bypass the policy: items carrying a story_id (vanilla quest-script convention), items the player gave to a companion (axr_companions), and player-strapped weapons (se_load_var). Player gifts and quest items survive trade through the categorizer itself. Everything else sells at the engine's cost field, half-price by default.

  Buy phase. The stalker walks the policy in declaration order and fills any category whose count falls below the min. Ammo resolves per equipped slot, consumables per category section set.

  Rank gates. Rookies buy basic ammo and a baseline of medkits and bandages. Veterans add premium ammo (AP, EP, PBP, hot-load, hollow-point, steel-core, flechette across 18 cartridges), grenades (F1, RGD5), and larger consumable bands. Veteran shots hit the player with the real k_ap of premium rounds.

  Profit cap. profit_max per rank caps net cash gain per trade event (rookie 1500 RU, veteran 5000 RU). Cash above the cap goes back to the trader so the till stays in business.

  Sell before buy. The sell phase runs before the buy phase, so cash from selling spares funds restocks. Squads with high-value spares restock more rows per visit instead of walking in cash-poor and out half-empty.

  This closes the in-Zone economy loop. NPCs harvest at anomaly fields, kill mutants for parts, loot or fill stashes. The surplus turns into cash at the next trader visit. The cash funds the ammo the next firefight burns through. The same loop the player walks, running for everyone.

Instincts

  Mutants have instincts.
  Drives are scored by deprivation, the same way as stalker needs.
  Scatter is binary and triggered by predator proximity.
  The strongest unmet instinct wins.
  - Scatter - Prey and lower-tier mutants scatter when they see a higher-tier predator within eye range.
    The food chain runs cowardly -> feral -> predator -> aberrant, with each tier fleeing all higher tiers.
    Squads relocate to the nearest smart terrain with no higher-tier threats.
  - Feed - Mutants move to open territory during active hours, where predators and prey meet on shared hunting grounds.
  - Sleep - Mutants return to rest locations during dormant hours: cowardly in fields, feral and predator in lairs, predator and aberrant in buildings.
  - Explore - Mutants wander to a different territory or lair during active hours.
  - Socialize - Pack animals move toward smart terrains where same-faction squads are present.

News: PDA radio gossip from the simulation

  Squad activity is published as radio chatter on your PDA. Every consequence (a squad sent to investigate a massacre, to set up an ambush, to scavenge bodies, to claim territory, etc.) rolls a per-consequence chance to publish. On hit, the event is captured to a session queue with the squad commander, faction, species, and location frozen at that moment.

  A composer drains the queue on a randomized interval and broadcasts one entry through Anomaly's PDA news channel.

  Each consequence ships with a pool of voice variants: hearsay (someone repeating it over the channel), eyewitness (someone who saw it), survivor (someone in it), tracker (someone reading the ground after), and named report (a factual line crediting the commander). The composer picks one at random, so the same consequence reads differently each time.

  Filters:
    - Level: only events on your current level pass.
    - Age: events older than the max-age window are dropped before broadcast.
    - Scope: own faction, allies, or world.
    - Channel: the speaker's faction is the radio channel. Monolith, Army, Greh, and ISG are member-only. Non-members never hear that chatter.
    - Vanilla rules: emissions and psi-storms suppress the queue. The PDA must be charged. The shared 3-message cap and faction-restricted channel rules apply.

  Sample radio lines:
    Heard a Free stalkers crew bedded down at Rookie Village a few hours ago after a long march.
    Tracks at Tent Camp show a Tushkanos pack ran out in every direction, and whatever drove them off is still there.
    Saw a Free stalkers crew walk into Rookie Village to ID their Bandit wipe with Wild Dogs already on the bodies.
    Heard a Bandit lost Trailer Camp to Military a few hours ago and the backup crew got there too late.

Day/Night Cycle

  The cycle gates which causes fire. Stalker needs and mutant instincts only run during the species' active phase. Sleep and shelter gate to dormant hours.
  Nocturnal species: bloodsuckers, lurkers, chimeras, zombies, fractures. Diurnal species: everything else.

---

For developers and advanced users:

Architecture:

- Radiant pipeline subscribes to squad-state events (squad updates, smart-terrain transitions, game-vertex changes). Each event passes a multi-gate chain (pacer, protection, ratio, budget) that admits a small handful per minute. Admitted events feed a cascade that shuffles registered radiant causes and stops on the first publish.
- Reactive pipeline subscribes to engine events that already encode "something happened": deaths, hits, item use, anomaly contact. Each registered cause evaluates independently. A single event can publish to many consequences.
- The simulation layer is the engine's own. The simulation board owns squad-to-smart routing, tracks which squads are at which smart, and fires the enter/leave callbacks other mods subscribe to. AlifePlus sets a one-shot destination override that replaces the target-picking step only, then clears on release.
- Smart terrain mutations (territory conquest, mutant infestation) rebuild from LTX on load. A scanner re-applies them and decays expired conquests on smart-terrain events.
- The runtime is xlibs, a reverse-engineered API wrapping the X-Ray engine source, built and validated against the C++ implementation.

Animations, Gulag, GOAP.

AlifePlus picks a destination smart terrain for a squad and hands the squad to the engine. From there the gulag binds each arriving NPC to a job from the smart terrain's catalog, scored by priority and precondition. Once bound, the scheme system loads the behavior module the job points to, registering its actions and evaluators on the NPC. The GOAP action planner then reads those evaluators each tick and picks the action that fits the NPC's current goal world state. AlifePlus works with that pipeline instead of replacing it.

Every cause and consequence is paired with a predicate that asks the engine which smart terrains can produce the corresponding activity. The predicate reads the same job catalog the gulag reads (smartcovers, patrol paths, animpoints per smart), with the same preconditions. That covers the vanilla Anomaly configs plus whatever modpacks like GAMMA/EFP/Zona add on top.

When a dispatch selects a destination, the engine routes the squad there through its normal systems. On arrival, the gulag selects the job, the scheme system resolves the behavior, and the action planner executes it. AlifePlus brokers between those systems with one goal: every smart terrain with real jobs ends up with NPCs running them. Anomaly ships a deep animation catalog that vanilla rarely exercises because most smart terrains never get occupied. AlifePlus closes that gap, and every animpoint, patrol path, and smartcover the game shipped finally runs in play.

If the engine cannot bind a job to every arriving squad member, the squad is released back into autonomous A-Life routing. The same release occurs when a precondition changes after arrival and the engine retracts the assigned job.

Off-map transit.

Any cause can flag a destination as off-map. The flag changes which smarts qualify and applies its own rate counter. The squad's arrival, hold, and release machinery is shared with on-map dispatch.

The engine carries the actual cross-level movement through its own routing slot. The gulag at the destination binds jobs the same way it does for on-map arrivals.

AlifePlus adds the protection layers around that capability:

- Per-source rate cap. Off-map dispatches are limited over a sliding game-hour window, so no single level keeps bleeding squads forever.
- Level adjacency. Cross-map dispatches stay within neighboring levels.
- Engine-exposed filtering. Cross-level scans run only on data the engine still exposes off-level: faction ownership and smart-terrain flags.
- Transit TTL. Every dispatched squad carries a recall timer if the engine never delivers it to the destination.
- Live squad-to-smart map. SIMBOARD's roster stays current at dispatch and release, so cross-level capacity and garrison checks read the right counts.
- Release on bind-fail. Arrival release fires if the destination cannot bind jobs to every member. Mid-hold release fires if the engine retracts a job at the destination after arrival.

Every off-map dispatch passes through the same broker that handles every on-map dispatch. The off-map flag affects only the selection and rate-limiting. Everything else is shared.

Performance:

Reactive. AlifePlus does no work when the engine fires no event.
Cheapest-first gates. A pure-Lua os.clock pacer rejects ~98% of the radiant ticks before any engine bridge runs. Protection, ratio, and budget gates follow in order.
Frame spreading. Long sweeps run through xslice across many frames via one shared AddUniqueCall driver, with deferred compaction (survivors swap) instead of per-item table.remove.
Cached over scanned. ap_core_cache holds per-level indexes for smarts, stashes, and squads. Smart and stash buckets never invalidate (LTX-baked). Squad buckets update incrementally on register / unregister, never rebuilt.
Smart data structures. xttltable provides O(1) token buckets with fractional accumulation, O(1) FIFO caches with ring-buffer eviction, and sliding-window TTL counters. Sorted lists use binary insert and binary search.
Minimal engine write surface. AP writes one engine field: squad.scripted_target via xsquad. Nothing touches npc_info, smart job tables, motivation_action_manager, or the scheme system.
Dual-clock rate limiting. Ephemeral pipeline limits read os.clock (zero luabind). Balance limits read xtime.game_sec (persisted across save/load, respects time acceleration and sleep).
C++ engine alignment. xlibs is built against the X-Ray engine source. Engine sentinels come from alife_space.h, online state reads C++ m_bOnline, protection guards match Anomaly's own sim_offline_combat layers. Every xlibs function carries an @cost annotation with luabind count and O() class.
Hard budget. Every measured flow targets 0.1ms average per call and a 4ms ceiling per call, including cold start and level transition.
Zero-overhead profiling. xprofiler and xtrace collapse to pre-allocated NOOP singletons below DEBUG. observe() drops to a ~150ns enabled-check. The hot path makes no allocations and no method calls.

[BENCHMARKS: screenshots side by side]

---

Compatibility & Safety:
- AlifePlus requires the latest Demonized or Demonized MT modded exes. Vanilla Anomaly and AOEngine are not supported.
- AlifePlus is built and tested with GAMMA, and also tested with Zona and EFP.
- The mod has no base script edits and no engine patches.
- Squads are extended, not hijacked: AlifePlus uses the engine's job system without overwriting engine variables.
- AlifePlus works mid-save.
- Story NPCs, companions, task givers, and quest squads are never touched.
- Squads owned by other mods (warfare, BAO) are excluded automatically via the ownership registry.
- Scripted squads carry a TTL and auto-release, so AlifePlus never holds a squad permanently.
- AlifePlus uses only engine-native mechanisms (the engine's scripted-target slot, the simulation board, the engine job system).
- AlifePlus does not need third-party "bridge" or "synergy" patches.
- Unauthorized patches that claim compatibility are not endorsed and will cause instability and save corruption.
- See integration.md on the project site for API reference and examples.

---

Alife Collection:
AlifePlus: https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01
AlifeBalance: https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance
AlifeGuard: https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001
AlifeTactics: TBD

---

Requirements:
- Anomaly 1.5.3
- demonized 20260601+ (https://github.com/themrdemonized/xray-monolith)
- xlibs (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
- MCM

Install (MO2):
1. Install xlibs
2. Install AlifePlus
3. Load order does not matter
4. Configure via MCM

Note: press RESET in MCM when updating. Most upgrade issues come from stale MCM state or outdated xlibs.

Uninstall (MO2):
Disable or remove in MO2.

---

Configuration:

Each cause and consequence is a module you can enable or disable through MCM.
Gameplay actions (trade, stash looting, supply trader visits) each have their own toggles and tunable values: chances, cooldowns, thresholds, quantities, rate limits, and budgets.
Log level goes from silent to full tracing with pathing, performance timing, and PDA map markers.

Two reset buttons live under Development. Reset ALL to Defaults restores every setting to factory state and takes effect on the first click. Reset ALL to Stateless Defaults restores factory state and then disables trade, item consumption, stash fill, stash loot, and smart ownership. AlifePlus dispatch and travel still run. Use the stateless preset on installs where another mod owns those systems.

Presets:
  Calm Zone: A-Life Interval 15-20, Cause Budget 5, Consequence Budget 1, Global Rate Limit 2
  Vanilla+: A-Life Interval 12, Cause Budget 10, Consequence Budget 2, Global Rate Limit 5 (defaults)
  Chaotic: A-Life Interval 2-4, Cause Budget 30+, Consequence Budget 5+, Global Rate Limit 15+

---

FAQ:

Do I need modded exes?
  Yes. AlifePlus requires the latest Demonized or Demonized MT modded exes. Vanilla Anomaly does not expose the APIs the framework relies on.

Does it work with other A-Life mods?
  AlifePlus has no known incompatibilities with warfare or AI addons.
  It will conflict behavior-wise with mods that script squads heavily.

Do I need offline combat enabled?
  AlifePlus does not require or prefer online simulation.
  It works with all offline settings, including fully disabled.
  Offline combat produces fewer combat events, but the system generates other events regardless, so A-Life activity continues.

---

Known Issues:
Map markers are debug-only and may glitch on some versions. Each marker shows the squad commander, so the stalker actually running the action may be a different squad member.

---

Validation:

luacheck and selene for static analysis, ast-grep for AST checks, contract rules (API safety, complexity, standards), lua54 integration tests against X-Ray engine stubs, gitleaks and trufflehog for secret scanning. Full report: doc/test-report.log.

Localization:
The mod includes English and Russian translations.

Credits:
Altogolik - support, ideas, source materials
Stalker_Boss - Russian translation
gwalls - English copyediting

---

Usage and License:
- Modpacks: allowed and encouraged. Keep the readme and license files.
- Addons, patches, integrations: allowed. Credit "AlifePlus by Damian Sirbu" visibly on your mod page.
- Reproducing the implementation in other software: not allowed, even with credit.
- Full license in LICENSE file and on GitHub.
