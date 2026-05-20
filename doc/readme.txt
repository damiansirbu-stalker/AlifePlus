AlifePlus: Emergent A-Life for STALKER Anomaly, by Damian
- Version: 1.6.0 (xlibs 1.5.1)
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
NPCs and mutants act independently, pursue goals, react to threats, and create interconnected effects.
Squads investigate massacres, hunt artefact carriers, claim empty territory, and act on hunger, sleep, and social needs.
Everything that happens to the player happens to NPCs and mutants alike.
Every action traces back to a cause, a world state, and a mechanic.

AlifePlus draws on Roadside Picnic and the original STALKER vision.
The Zone runs on its own rules, and the actor is just another entity inside it.
Each faction acts on its identity: Duty holds ground, Bandits ambush and loot, Ecologists research, Monolith never retreats.
Alignment determines what a faction can do at all, and personality determines how likely it is to act.
The manifesto carries the source references.

Every action has a systemic cause. The simulation runs whether you are there or not.
Structural invariants make this possible:
- Event-driven contract: nothing runs unless the engine says something happened.
- Physical simulation guarantee: consequences use entities already in the simulation, never spawned, teleported, or fabricated.

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
- NPCs walk to stashes to loot, ambush, or fill them with supplies.

Territory and population:
- Stalkers and mutants claim empty outposts.
- Conquests decay if nobody holds them.
- Mutants infest lairs and buildings as nests.
- Radiant dispatches weigh tactical soundness. A squad that would leave its base undefended by its faction, or pile onto a destination another scripted squad already targets, scores low and usually stays.

Alpha mutants:
- Alpha mutants emerge from combat and accumulate kills.
- Alphas gain hit power buffs and drop valuable loot.

Trade and needs:
- NPCs trade real items at traders.
- Stalkers swap artefacts for ammo, grenades, or medkits.
- Hunger, fatigue, and social needs drive campfire and base behavior.

Day/night cycle:
- Stalkers work and trade by day.
- Stalkers rest and sleep at campfires by night.
- Nocturnal predators hunt at night and retreat to lairs at dawn.
- Diurnal mutants follow the opposite cycle.
- Prey scatters when a predator moves through.
- Prey returns once the predator dies.

Chained and emergent behavior:
- One event triggers another consequence, which triggers another, with no hand-scripted sequence behind any of it.
- Each faction responds to the same event according to its personality, not random chance.

Faction identity:
- Bandits loot and ambush.
- Duty patrols and holds the line.
- Loners chase artefacts.
- Military stays close to base.
- Monolith pushes into territory.
- Renegades scatter at first contact.

---

Example scenario (systemic interaction):

- A stalker reaches a location, and AlifePlus intercepts that event.
- He spots a stash ("stash" cause) and his faction's greed is high enough to loot it ("loot" consequence).
- He walks to the stash, but a bandit who saw it first ambushes him ("ambush" consequence).
- The player was heading to that stash, sees the fight, and snipes both of them.
- This mass killing ("massacre" cause) draws cowardly mutants to the bodies ("scavenge" consequence).
- The victims' faction sends squads to investigate ("investigate" consequence).
- On the way there, loners out hunting kill the scavengers ("job" consequence).
- Those loners are now tired ("needs" cause) and head back to base to smoke a cigarette and tell the story ("social" consequence).
- None of this was scripted.
- The player walked into something already in motion.

Example scenario (escalation chain):

- An artefact spawns. A stalker needs money ("needs" cause) and heads there to pick it up ("money" consequence).
- He fights creatures on the road and they fight back.
- A chimera accumulates kills and becomes an alpha ("alpha promote" consequence).
- The alpha hits harder, takes less damage, and never flees.
- The player wounds it but it keeps fighting because of panic immunity.
- He kills it and finds valuable mutant parts and an artefact in its inventory.
- Another chimera on the same tier senses the kill and pursues the player ("alphakill targeted" consequence).
- An online stalker on a friendly channel reports the alpha down a few hours later as a news event on the player's PDA.
- A surviving chimera with its pack claims a nearby smart terrain as a nest ("infest" consequence). The infestation overwrites the original spawns.
- A bandit squad sees the ground is empty around the lair and conquers a neighboring smart ("area_conquer" consequence). Their spawns join the originals as a shared injection.
- The cycle continues. The Zone does not care.

---

Why this mod exists:

A-Life modding is hard.
The engine exposes limited APIs, documentation is scarce, and most knowledge comes from trial and error.

Common patterns scan the world every frame, permanently hijack squads by overwriting engine variables, swallow crashes silently, and accumulate stale state across saves with no cleanup.
The result is poor performance, entity leaking, ghosting, save corruption, and mod conflicts that get worse the longer you play.

---

The AlifePlus Approach:

Engine events are intercepted as they fire.
The system reacts when something happens, not on a timer.
AlifePlus is idle when no events fire.

Squads are extended, not hijacked.
The engine's own job system produces the correct behavior without overwriting its variables.

Events chain deterministically.
Causes dispatch consequences.
Consequences change the world state.
Each change triggers new causes.
Behavior is the result of independent actors colliding in a shared simulation.

Cause families:
- Reactive causes: triggered by combat, wounds, deaths, or item use.
- Radiant causes: triggered by movement, location, and needs.

AlifePlus is first and foremost a framework.

Core and ext are separated by a hard rule.
Core is the pipeline: gates, protection, rate limiting, tracing, squad lifecycle, and PDA routing.
Ext is the domain: causes, consequences, alignment, personality, news, mutant infestation, and territory conquest.
Core never imports ext. All domain logic reaches the framework through registered function references.

Three integration levels are open to other mods:
- Listen: subscribe to framework events and read cause data. xlibs only, no AP code dependency.
- Register: add a cause predicate and a consequence handler. Inherit protection, rate limiting, tracing, PDA routing, and squad lifecycle for free.
- Coordinate: use the tracker, conquest, broker, or news API for territory control, ownership, or message dispatch.

The pipeline is rate-bounded and waste-free.
A pacer + protection + ratio + budget chain reduces the engine's raw squad-update volume to a small handful of triggers per minute.
Every triggered evaluation produces a result.

Protection runs at the producer, cause, consequence, and squad layers.
Story NPCs, task givers, companions, externally scripted squads, and squads owned by other mods (warfare, BAO via the ownership registry) are excluded.

New causes and consequences plug in without modifying core code.
A consequence describes what should happen, not how to plumb it.
The framework handles the rest.

Other mods integrate at any level without conflict.
AlifePlus uses only engine-native mechanisms (the engine's scripted-target slot, the simulation board, the engine job system).
Any mod that respects the engine's own APIs will not conflict.
See integration.md on the project site for examples, API reference, and a concrete two-mod collaboration scenario.

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
  The behavioral tier determines what a species does. The activity axis determines when it acts.
  The tiers form a food chain in which lower tiers scatter from higher tiers, with aberrant species at the top.

Personality (probability layer):

  Per-faction and per-species trait profiles, with values derived from GSC's AI design documents.
  Stalker profiles carry these traits: aggression, greed, survival, perception, territory, relation, and discipline.
  Mutant species profiles carry these traits: aggression, survival, territory, perception, and relation. Animals do not read greed or discipline.
  Some traits (aggression, territory, discipline) have inverted variants for consequences that fire on low scores. A flee consequence reads inverted aggression so a low-aggression faction is the one most likely to flee.
  Each consequence binds to its relevant traits and averages them.
  The averaged value is clamped to a configurable floor and ceiling (configurable in MCM) and used as the probability of acting.
  Trait values are used directly as probabilities.

World state (scan layer):

  The scan phase queries live world state before the consequence acts.
  Smart terrain capacity, ownership, current engine-native squad control, recent activity, faction territory bookkeeping, time of day, online/offline status.
  Stale or contradictory state rejects the consequence. A squad already scripted by another mod is left alone. A full smart terrain rejects new arrivals. An emission or psi-storm suppresses the news queue.

---

Features:

Causes and consequences aggregate into hundreds of combinations.

Reactions

  World events trigger responses.
  Consequences search within radio or scent range.

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
    - Promote - Mutants accumulate kills and level up.
      Hit power bonus and reduction scale with level.
      Promoted alphas gain panic immunity.
      Valuable loot is injected into the alpha's inventory on promotion.

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
  Consequences search within eye range.
  A per-squad drive cooldown prevents repeated firing.

  Stash
    - Loot - A stalker squad spots the stash, walks there, and loots it.
    - Ambush - An outlaw squad spots the stash and camps it as bait.
    - Fill - A stalker squad spots the stash and hides supplies inside.

  Area
    - Conquer - Stalker factions claim empty territory (Ecologists excluded; mutants use Swarm).
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
  Consequences search within radio range.
  - Hunger - The stalker finds a campfire and eats what he is carrying (bread, sausage, canned goods).
  - Sleep - The stalker finds a campfire during dormant hours and sleeps.
  - Rest - The stalker finds a campfire, smokes a cigarette, and has a drink.
  - Heal - The stalker finds a safe location and uses a medkit, bandage, or stimpack.
  - Shelter - The stalker finds a safe location when exposed too long.
  - Money - The stalker searches anomaly fields for artefacts or hunts mutant lairs.
  - Supply - The stalker visits a trader and trades an artefact for AP ammo, grenades, or medical supplies.
  - Job - The stalker guards outposts and checkpoints, explores the Zone, or researches anomalies.
  - Social - The stalker finds a campfire or safe location and shares cigarettes and drinks.

  Cross-map sibling rows: Supply, Job Explore, and Social each ship a paired off-map row. Greedy nomads (Loners, Bandits, Mercs, Renegades) chase trade. Peaceful curious factions (Ecologists, Clear Sky) scout neighbor maps. Peaceful sociable factions (Ecologists, Freedom, Loners) visit other camps. Army, Monolith, and Zombified squads never travel off-map.
  Reach grows with player progression. Stalkers reach the next map once Yantar X-16 is shut down, the next after the Brain Scorcher is deactivated at Radar, and master-rank squad commanders push one map further. Each step is tunable under MCM > World > Off-map.
  Squads return home after a few in-zone days. Travellers stuck off-map are released after a week to keep level populations clean. Dispatches are also capped per source map within a sliding two-day window so no level bleeds traffic forever.

  NPCs consume real items from their inventory on arrival.
  A guard smokes a cigarette on duty.
  Stalkers go to traders to swap artefacts for ammo, grenades, or medical supplies.

Instincts

  Mutants have instincts.
  Drives are scored by deprivation, the same way as stalker needs.
  Scatter is binary and triggered by predator proximity.
  The strongest unmet instinct wins.
  Consequences search within eye or scent range.
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
    - Channel: the speaker's faction is the radio channel. Monolith, Army, Greh, and ISG are member-only; non-members never hear that chatter.
    - Vanilla rules: emissions and psi-storms suppress the queue; the PDA must be charged; the shared 3-message cap and faction-restricted channel rules apply.

  Sample radio lines:
    Heard a Free stalkers crew bedded down at Rookie Village a few hours ago after a long march.
    Tracks at Tent Camp show a Tushkanos pack ran out in every direction, and whatever drove them off is still there.
    Saw a Free stalkers crew walk into Rookie Village to ID their Bandit wipe with Wild Dogs already on the bodies.
    Heard a Bandit lost Trailer Camp to Military a few hours ago and the backup crew got there too late.

Day/Night Cycle

  Stalkers and mutants follow a day/night activity cycle that gates the entire simulation rhythm.
  Stalkers are active during the day and sleep at night.
  Nocturnal mutants reverse the cycle: bloodsuckers, lurkers, chimeras, zombies, and fractures.
  During active hours, creatures feed, explore, work, and trade. During dormant hours, they seek shelter and sleep.
  Stalker needs and mutant instincts only fire during the species' active phase, except sleep and shelter which gate to dormant hours.
  Predators hunt during their active window, and prey returns to common ground while predators rest.
  Day belongs to trade and territory; night belongs to hunting, scavenging, and rest; a handful of species keep their own clock.
  Daytime in a level is human movement and trade.
  Nighttime is mutant movement and predation.

---

For developers and advanced users:

Architecture: Purely reactive.

The framework holds no scheduler. There is no clock, no polling loop, no per-frame work. Every evaluation is the direct consequence of an engine event the framework subscribed to; when no event fires, nothing runs.

- Radiant pipeline subscribes to squad-state events (squad updates, smart-terrain transitions, game-vertex changes). Each event passes a multi-gate chain (pacer, protection, ratio, budget) that admits a small handful per minute. Admitted events feed a cascade that shuffles registered radiant causes and stops on the first publish.
- Reactive pipeline subscribes to engine events that already encode "something happened": deaths, hits, item use, anomaly contact. Each registered cause evaluates independently; a single event can publish to many consequences.
- No base-script edits. No engine patches. Only runtime callbacks and hooks.
- The simulation layer is the engine's own. The simulation board owns squad-to-smart routing, tracks which squads are at which smart, and fires the enter/leave callbacks other mods subscribe to. AlifePlus sets a one-shot destination override that replaces the target-picking step only, then clears on release.
- Smart terrain mutations (territory conquest, mutant infestation) rebuild from LTX on load. A scanner re-applies them and decays expired conquests on smart-terrain events.
- The runtime is xlibs, a reverse-engineered API wrapping the X-Ray engine source, built and validated against the C++ implementation.

Animations, Gulag, GOAP.

AlifePlus picks a destination smart terrain for a squad and hands the squad to the engine. From there the gulag binds each arriving NPC to a job from the smart terrain's catalog, scored by priority and precondition. Once bound, the scheme system loads the behavior module the job points to, registering its actions and evaluators on the NPC. The GOAP action planner then reads those evaluators each tick and picks the action that fits the NPC's current goal world state. AlifePlus works with that pipeline instead of replacing it.

Every cause and consequence is paired with a predicate that asks the engine which smart terrains can produce the corresponding activity. The predicate reads the same job catalog the gulag reads (smartcovers, patrol paths, animpoints per smart), with the same preconditions. That covers the vanilla Anomaly configs plus whatever modpacks like GAMMA/EFP/Zona add on top.

When a dispatch selects a destination, the engine routes the squad there through its normal systems. On arrival, the gulag selects the job, the scheme system resolves the behavior, and the action planner executes it. AlifePlus acts as a broker between those systems, integrating with the engine's existing decision flow.

If the engine cannot bind a job to every arriving squad member, the squad is released back into autonomous A-Life routing. The same release occurs when a precondition changes after arrival and the engine retracts the assigned job.

Off-map transit.

Any cause can flag a destination as off-map. The flag changes which smarts qualify and applies its own rate counter. The squad's arrival, hold, and release machinery is shared with on-map dispatch.

The engine carries the actual cross-level movement through its own routing slot. The gulag at the destination binds jobs the same way it does for on-map arrivals.

AlifePlus adds the protection layers around that capability. A per-source-level rate counter limits off-map dispatches over a sliding game-hour window, so no single level keeps bleeding squads forever. The destination scan respects level adjacency, so cross-map dispatches stay within neighboring levels. Cross-level filtering runs only on data the engine still exposes off-level: faction ownership and smart-terrain flags. Every dispatched squad carries a transit TTL that recalls it if the engine never delivers it to the destination. The engine's squad-to-smart map stays current at dispatch and release, so cross-level capacity and garrison checks read the right counts. The arrival release fires if the destination cannot bind jobs to every member, and the mid-hold release fires if the engine retracts a job at the destination after arrival.

Every off-map dispatch passes through the same broker that handles every on-map dispatch. The off-map flag affects only the selection and rate-limiting. Everything else is shared.

Performance:

Performance was the first design constraint. Performance impact stays at near-zero in normal play.

AlifePlus is reactive by architecture: it does no work when nothing happens. Nothing scans the engine world, polls engine state, or hunts for work each frame. The only loops are internal cleanup over the framework's own bookkeeping, run on coarse time events. The framework wakes when the engine fires a callback, runs through cost-ordered gates, dispatches via pub/sub to consequence handlers, and returns. Every operation is timed; heavier maintenance is sliced across many frames so no single tick spikes.

AlifePlus only calls the engine the way the engine expects to be called. Every entry point mimics or continues what X-Ray and Anomaly already do; every invariant, flag, and design rule of the engine is respected; nothing is bypassed or reimplemented to shortcut the engine's own model.

Bridge crossings are the scarce resource. Every xlibs function is annotated with its real cost: complexity, engine-bridge count, and C++ weight per crossing. Pure Lua is the default, protected calls are reserved for the few approved recovery categories, and the reject side of every hot-path gate allocates nothing. Helpers were rewritten directly against the X-Ray engine source with zero-allocation paths instead of bridge-heavy ones.

Cost-ordered gates strip the bulk of callback volume with cheap rejects (pure Lua, no engine bridge) before any expensive check runs.

Long sweeps amortize across many frames via one shared driver. Cheap predicates (id, faction, level) run first; spatial queries use squared distance and hash-keyed engine tables.

Profiling is built into the pipeline rather than bolted on. Every span is timed via the engine's microsecond timer and reported per minute. Below DEBUG, the profiler and tracer collapse to no-op stubs with no allocation on the hot path.

[BENCHMARKS: screenshots side by side]

---

Compatibility & Safety:
- AlifePlus is tested with vanilla Anomaly 1.5.3, Demonized main, Demonized MT, and AOEngine.
- AlifePlus is built and tested with GAMMA, and also tested with Zona and EFP.
- The mod has no base script edits and no engine patches.
- AlifePlus works mid-save.
- Story NPCs, companions, task givers, and quest squads are never touched.
- Squads owned by other mods (warfare, BAO) are excluded automatically via the ownership registry.
- Scripted squads carry a TTL and auto-release, so AlifePlus never holds a squad permanently.
- Other mods can integrate at any level.
- They listen to events, register new behaviors, or coordinate squad control.
- AlifePlus uses only engine-native mechanisms (the engine's scripted-target slot, the simulation board, the engine job system).
- AlifePlus does not need third-party "bridge" or "synergy" patches.
- Mods that adopt the AP framework through the official API are supported.
- Unauthorized patches that claim compatibility are not endorsed and will cause instability and save corruption.
- See integration.md on the project site for API reference and examples.

---

Companion mod:

AlifePlus increases NPC activity and combat across the Zone. AlifeBalance reads those death events and accelerates respawn cooldowns on the level and faction where losses happened. Heavy combat regions recover, quiet ones stay quiet.
AlifeBalance: https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance

---

Requirements:
- Anomaly 1.5.3
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
Gameplay actions (item consumption, stash looting, artefact trading, rank progression) each have their own toggles and tunable values: chances, cooldowns, thresholds, quantities, rate limits, and budgets.
Log level goes from silent to full tracing with pathing, performance timing, and PDA map markers.

Presets:
  Calm Zone: A-Life Interval 15-20, Cause Budget 5, Consequence Budget 1, Global Rate Limit 2
  Vanilla+: A-Life Interval 12, Cause Budget 10, Consequence Budget 2, Global Rate Limit 5 (defaults)
  Chaotic: A-Life Interval 2-4, Cause Budget 30+, Consequence Budget 5+, Global Rate Limit 15+

---

FAQ:

How do I make events occur faster or slower?
  Calmer world: raise A-Life Interval to 8-10, lower Consequence Budget to 1, raise Radiant Cooldown to 15-20.
  Busier world: lower A-Life Interval to 2-3, raise Consequence Budget to 4, lower Radiant Cooldown to 5.
  Gung-Ho: lower A-Life Interval to 1, raise Cause and Consequence Budget to max, set Radiant Cooldown to 0. Good luck!

Does it conflict with the vanilla task system?
  AlifePlus validates every entity through protection gates at every pipeline layer.
  Task givers, companions, active quest squads, and externally scripted squads are protected.

Does it work with GAMMA?
  AlifePlus is built and tested with GAMMA, and also tested partially with Zona and EFP.
  It works directly with X-Ray engine callbacks without modifying base scripts.

Does it work with other A-Life mods?
  AlifePlus has no known incompatibilities with warfare or AI addons.
  It will conflict behavior-wise with mods that script squads heavily.

Does it work mid-save?
  Yes.

Do I need offline combat enabled?
  AlifePlus does not require or prefer online simulation.
  It works with all offline settings, including fully disabled.
  Offline combat produces fewer combat events, but the system generates other events regardless, so A-Life activity continues.

---

Known Issues:
Map markers are a debugging feature, may glitch in various versions and i sugegst to not use them as game aids.

---

Validation:
Multi-stage validation pipeline:
- luacheck and selene (static analysis)
- tree-sitter AST analysis with ast-grep structural patterns
- Contract rules (API safety, cross-file dependencies, cyclomatic complexity, coding standards)
- lua54 integration testing with X-Ray engine stubs
- gitleaks and trufflehog (secret scanning)
The full report lives in doc/test-report.log.

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
