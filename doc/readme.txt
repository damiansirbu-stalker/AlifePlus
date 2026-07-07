AlifePlus: Emergent A-Life for STALKER Anomaly, by Damian
- Version: 1.8.1 (xlibs 1.8.0)
- Manifesto: https://github.com/damiansirbu-stalker/AlifePlus/blob/main/doc/manifesto.md
- Integration guide: https://github.com/damiansirbu-stalker/AlifePlus/blob/main/doc/integration.md
- Changelog: https://github.com/damiansirbu-stalker/AlifePlus/blob/main/doc/changelog
- Russian / На русском: https://github.com/damiansirbu-stalker/AlifePlus/blob/main/doc/readme_ru.txt
- Bugs, suggestions: https://github.com/damiansirbu-stalker/AlifePlus/issues

Alife Collection:
AlifePlus: https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01
AlifeBalance: https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance
AlifeGuard: https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001
AlifeTactics: https://www.moddb.com/mods/stalker-anomaly/addons/alifetactics

! Please use the RESET button in MCM when updating to a new version !

You are not special.

AlifePlus is a reactive A-Life framework for STALKER Anomaly.
It extends A-Life with event-driven emergent behavior, built on GSC's original AI design.
It intercepts engine events, classifies them into causes, and dispatches consequences that extend the simulation.
The X-Ray engine exposes the primitives for emergent behavior: a callback for every world event, a graph the simulation walks, a server object per entity.
Stock Anomaly spends them on a random movement generator: the weights sit on the smart terrains, identical for every squad, so no squad decides for itself.
AlifePlus spends the same primitives but adds events, world state, and per-squad business logic grounded in GSC's design: alignment, personality, Hull drives, marginal value theorem.
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
- Money flows match vanilla: NPCs gain money when they sell items at supply traders, spend money when they buy ammo and consumables. No money moves through stash loot, stash fill, the faction market, or any other AP flow. The market only adds trader stock and sets its price, which you pay at the trader like any purchase.

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

Loot ownership:
Every kill belongs to whoever made it, the player and the stalkers alike. The two sides toggle independently in MCM.
- NPCs no longer strip the kills you made while you stand nearby. Walk away and scavengers may take them.
- A body a living NPC killed will not open while that stalker is alive and near it. Walk off, or wait until he dies, and it opens up.
- NPCs keep only useful loot from a body: ammo for their own weapon and a capped share of each supply, and the rest is destroyed, so bodies you reach later are not stripped bare.
- These controls and the per-visit trade profit cap live under the Economy menu.

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

  Territory
    - Area Conquer - Stalker factions claim empty territory (Ecologists excluded). Mutants use Area Swarm.
      Conquest adds the conqueror's spawns alongside the originals as a shared injection (both streams stay active).
      Territory decays over time (MCM configurable).
    - Area Swarm - Feral, predator, and aberrant mutant squads claim empty smart terrains.
      Swarming adds the swarming species' spawns alongside the originals as a shared injection (both streams stay active).
      Territory decays over time.
      Decay, cap, and spawn count tune independently from Area Conquer in MCM.
    - Area Infest - Feral, predator, and aberrant mutants claim smart terrains as nests.
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
  Squads settle at the destination after a short stay; they do not return home. Travellers stuck off-map are released after a week to keep level populations clean. Dispatches are also capped per source map within a sliding two-day window so no level bleeds traffic forever.

  Arrival satisfies the need. Inventory items are not destroyed by the arrival itself. Engine-side combat behavior (medkit, bandage, stim usage in firefights) is independent and unchanged.

Economy

The Zone runs one connected economy, and with the faction market it now comes full circle. Four systems share a single set of category rules, so what a stalker loots, keeps, sells, and stashes all answer to the same limits, and a slice of what a faction sells eventually circles back to you.

  Loot claim. A kill belongs to whoever made it, held while the owner stays near the body. The same rule runs three ways, identical for every side: your kills against NPC looters, an NPC squad's kills against you, and rival squads against each other.

  Loot policy. An NPC keeps ammo that fits its own weapons and the most valuable items in each class, then destroys the rest. Stalkers carry a working loadout, the bodies you reach are leaner, and long saves stay clear of the loot buildup that drags them down.

  Trade. At a trader a stalker sells his surplus and restocks ammo and gear by rank, under a profit cap. The same category limits decide what he offloads and what he buys.

  Market. A faction's hub traders briefly stock a bounded echo of what that faction's stalkers recently sold, gated to your rank and priced at a premium. The surplus a faction sells reaches you through its own traders.

Loot Claim

  A kill is reserved for whoever made it, against every other looter, while the owner is present and the claim is fresh. An NPC kill belongs to the killer's squad: any living squadmate within range holds it, so dropping one stalker does not free the body while the rest of his squad stands over it. This is a three-way reservation, the same rule running in all three directions, each its own toggle, radius, and duration, all under Economy > Loot claim.

  You against NPCs. A body you killed is skipped by NPC looters while you stand within the reserve radius. The scavenger AI treats your kill as off-limits and walks past it. Step away and the reservation lapses, so a passing stalker may strip it like any other corpse.

  NPCs against you. A body a stalker squad killed will not open while a living member of that squad is within range of it. Aim at it, press use, and nothing happens but a PDA tip naming the owner. When the squad is dead or beyond the radius, or the claim runs out its hour, the body opens and loots like any other. Companions, zombified killers, and story corpses never hold a claim against you.

  NPCs against each other. A passing stalker will not strip a kill another squad made while a member of that squad is near the body. The killing squad collects its own. Bystanders walk past. The loot a squad earns stays with that squad until they leave or fall.

  Radius and duration. Each side carries its own reserve distance and its own claim lifetime, all 150 m and 1 game hour by default, tunable under Economy > Loot claim. Beyond the radius, or once the lifetime elapses, the claim lapses. Turn all three off for plain vanilla looting in every direction.

Loot Policy

  An NPC that loots a body keeps only what is useful to it and destroys the rest, so the bodies you reach later are leaner instead of picked clean by passing stalkers. Only the fresh pickup is bounded. The looter's standing gear is left alone.

  Kept: ammo that fits the looter's own pistol or rifle, plus a capped share of each supply class (a few medkits and bandages, some food and drink, one spare weapon, one outfit, and so on). Ammo for weapons the looter does not carry is dropped. When a class runs over its cap, the cheap surplus is what goes and the most valuable items stay.

  Quest items, story gear, an item you gave a companion, and the looter's own equipped weapon and armour are never touched. Companions, traders, and named characters keep their loot untouched.

  The caps live in an editable policy file, and the system toggles under Economy > Loot policy.

Trade

  AlifePlus runs a category-based NPC buy/sell cycle on arrival at trader smarts. NPCs visit a trader, sell surplus from their inventory, and restock ammo, grenades, and consumables per a per-rank policy. All 20 vanilla trader smarts are covered, from Sidorovich at Cordon to the Monolith trader in Pripyat. Built on Alundaio's buy/sell core (2013, maintained by Tronex through 2019), modernized to accept extension instead of dirtying item files, and to carry depth the original row schema could not express.

  Categories instead of items. Alundaio's row schema lived inside each tradeable item's LTX: every section had to declare its own buy_sell row, and modpacks adding items had to dirty the item files to participate in trade. AlifePlus declares categories in a DLTX-overridable policy file. xinventory resolves any section to its category through the engine's own item buckets. A Boomsticks AP round, a GAMMA addon medkit, an EFP grenade all classify automatically through their existing kind and class fields. The modder adds the item, the mod ships, AlifePlus trades it. No per-modpack LTX maintenance, no item file edits.

  Per-rank policy. Two blocks split policy by character rank: rookie (rank below 12000) and veteran (rank 12000 and above). Each block lists categories with a min, max band: medkit, bandage, antirad, stim, pill, grenade, food, drink, plus per-slot ammo tiers (ammo_slot_2_t1/t2 pistol basic/premium, ammo_slot_3_t1/t2 rifle basic/premium). Entry order sets the buy priority.

  Real equipped weapons drive ammo picks. The pistol's ammo class is read at runtime, sorted by k_ap, split into basic and premium tiers. The rifle's ammo class splits the same way. A veteran with an SVD gets basic and premium 7.62x54R. A rookie with a PM gets basic 9x18 only. Within a tier, the cheapest section is picked first so NPCs with limited cash actually fill their slot. Consumables walk cheapest-first from the category's section set.

  Sell phase. The stalker walks his inventory and drops anything whose category count exceeds the policy max. Equipped items (whatever sits in his slots, knife to backpack), quest items, anim items, and money are never touched. Three runtime untouchables also bypass the policy: items carrying a story_id (vanilla quest-script convention), items the player gave to a companion (axr_companions), and player-strapped weapons (se_load_var). Player gifts and quest items survive trade through the categorizer itself. Everything else sells at the engine's cost field, half-price by default.

  Buy phase. The stalker walks the policy in declaration order and fills any category whose count falls below the min. Ammo resolves per equipped slot, consumables per category section set.

  Rank gates. Rookies buy basic ammo and a baseline of medkits and bandages. Veterans add premium ammo (AP, EP, PBP, hot-load, hollow-point, steel-core, flechette across 18 cartridges), grenades (F1, RGD5), and larger consumable bands. Veteran shots hit the player with the real k_ap of premium rounds.

  Profit cap. One cap limits the net cash an NPC gains per trade event, the same value for every rank, 5000 RU by default. Cash above the cap goes back to the trader so the till stays in business.

  Sell before buy. The sell phase runs before the buy phase, so cash from selling spares funds restocks. Squads with high-value spares restock more rows per visit instead of walking in cash-poor and out half-empty.

  This closes the in-Zone economy loop. NPCs harvest at anomaly fields, kill mutants for parts, loot or fill stashes. The surplus turns into cash at the next trader visit. The cash funds the ammo the next firefight burns through. The same loop the player walks, running for everyone.

Market

  The market is the read side of the trade cycle. The trade cycle above moves items from stalkers into traders. The market records what passes through and stocks a rank-gated slice of it back at those traders for you to buy.

  What you see. When a faction's hub trader restocks, a few of the items that faction's stalkers recently sold appear in his stock, gated to your rank, priced at a premium, and gone again after a short window.

  The ledger. Every NPC sale at a trader is recorded by faction in a bounded list of recent sales. What a faction sells, that faction's traders may later carry. The list keeps only the most recent distinct items and drops the oldest as new ones come in, so the stock reflects recent sales. Nothing leaves the trade cycle to feed this. The market only reads it.

  The echo. A trader's stock is wiped and respawned at every restock, so the item a stalker sold him is already gone by the time the market acts. The market re-creates a bounded echo of that sale, never the original item. Nothing is duplicated and the trade economy is left untouched.

  Rank gate. What appears is gated by your rank, not the NPC's. Cheap goods show up early, expensive ones only as you rise. A rookie sees common consumables and parts, a veteran the high-value artefacts, devices, and medical a faction is losing. The gate, the allowed items, and their caps live in an editable policy file. Traders sell consumables, artefacts, devices, and grenades, so the market echoes those. Gear like outfits, helmets, and weapons is not stocked, since traders do not sell it.

  Price and condition. An item shows at one trader, once, for a short window, at several times its full value, five by default. Different traders of a faction carry different items. The premium and the spawn condition are sliders under Economy then Faction market.

  Adds only. The market never takes money or items from anyone. It adds stock and sets the price, and you pay that premium through the normal trade window like any other purchase. Turn it off and traders carry only their vanilla stock. It ships on.

  What a faction loots it keeps by policy, what it keeps it carries and sells, and what it sells circles back through its traders to you. The stock is never more than what that faction actually sells. The coarse controls live under Economy then Faction market. The detail lives in the policy file.

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

- Radiant pipeline subscribes to ap_squad_on_change, a synthetic callback AlifePlus produces that watches every squad in the Zone and produces events at exactly the configured rate: squads that really changed (position, smart, online state, movement) fire first, quiet squads take round-robin turns, and the A-Life Ratio splits the rate between your map and background maps at the source. Each fire passes a gate chain (dispatch budget, protection) and feeds a cascade that shuffles registered radiant causes and stops on the first publish.
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

The engine carries the actual cross-level movement through its own routing slot. The gulag at the destination binds jobs the same way it does for on-map arrivals. There is no return leg: the squad holds the destination gulag for a short stay and then settles there as an ordinary resident. Its home becomes the destination.

AlifePlus adds the protection layers around that capability:

- Per-source rate cap. Off-map dispatches are limited over a sliding game-hour window, so no single level keeps bleeding squads forever.
- Level adjacency. Cross-map dispatches stay within neighboring levels.
- Engine-exposed filtering. Cross-level scans run only on data the engine still exposes off-level: faction ownership and smart-terrain flags.
- Despawn safety net. A squad the engine never delivers, or that cannot settle at the destination, is despawned when it is offline so it never lingers off-map.
- Live squad-to-smart map. SIMBOARD's roster stays current at dispatch and settle, so cross-level capacity and garrison checks read the right counts.
- Release on bind-fail. Arrival release fires if the destination cannot bind jobs to every member. Mid-hold release fires if the engine retracts a job at the destination after arrival.

Every off-map dispatch passes through the same broker that handles every on-map dispatch. The off-map flag affects only the selection and rate-limiting. Everything else is shared.

Vanilla fixes:

AlifePlus corrects two long-standing vanilla Anomaly A-Life bugs. Both are wrapped rather than replaced, so they layer cleanly on any base game or modpack and go inert where a modpack already fixes the same thing.

- Squad chase. Vanilla cannot point one offline squad at another as a moving target. AlifePlus rebuilds the pursuit each tick, so NPC-versus-NPC and NPC-versus-player chases track the target's real position instead of a stale one.
- Smart terrain headcount. When the engine moves a scripted squad between smart terrains, vanilla forgets to update that terrain's squad count. Capacity checks then drift, and a terrain can read as full when it is not. AlifePlus keeps the count correct for every squad move, its own and the engine's.

These patches are global and affect every squad, not only AlifePlus's. If another mod already patches the same engine scripts, disable ap_core_chase.script and ap_core_anomaly_fixes.script before installing.

Performance:

Reactive. AlifePlus does no work when the engine fires no event.
Cheapest-first gates. Radiant events arrive pre-shaped: the xlibs squad sweep visits 20 squads per second at constant cost and produces exactly the configured events per minute, replacing the raw engine heartbeat of ~6,600 calls per minute. Only a budget read and protection follow.
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
- Requires xlibs.
- Runs on themrdemonized modded exes 2025.9.10 or newer, or AOEngine v0.55 or newer.
- The full feature set needs the latest demonized build. A feature that needs a newer build stays inactive on older exes.
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

Mod compatibility:

  Superseded:
  - NPC Loot Claim, NPC Loot Claim Remade: loot ownership covers all three directions (your kills, NPCs' kills, and between NPCs). Disable them, otherwise both intercept looting a claimed corpse and fight over it.
  - Anti-loot addons (NPC Stop Looting Dead Bodies, BoltBeGone): the loot policy keeps NPC looting on and bounds what each looter keeps, so blocking the loot path is no longer needed. AlifeBalance's Inventory Balance does the same for standing inventory.

  Conflicts (critical):
  - Unauthorized "synergy" / "bridge" patches claiming to connect AlifePlus to another mod: instability, save corruption.
  - Squad-scripting "brains" that take squads without the ownership handshake: two systems fight for the same squads.

  Affects / coexists:
  - Warfare A-Life Overhaul, Better A-Life Overhaul: own their squads; excluded via the ownership registry.
  - Faction-relation mods (Dynamic Faction Relations, zone-relation tweaks): change which causes can fire. Intended.
  - Spawn / population mods (Dynamic Mutants, ReSpawn Mutant Collection, Rebound Encounters): more entities to act on; pair with AlifeGuard.
  - Extended sim-distance / offline mods (Living Zone, Extended Offline, ROAD alife range): enlarge the population; fight AlifeGuard more than AlifePlus.
  - Autolooter / proximity-loot mods: opening a claimed corpse is blocked, so auto-loot driven through the loot window is stopped with it. Only remote/proximity transfer that never opens the corpse can still take a claimed kill. Reserving your own kills from NPC looters is unaffected.
  - Interaction Dot Marks (Catspaw): opening, take-all, and auto-loot of a claimed corpse through its dot are blocked. Remote transfer that never opens the corpse can still take a claimed kill.
  - Death-drop / weapon-on-ground mods: gear dropped to the ground on death is taken through the gather-items scheme, outside corpse ownership.

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
Gameplay actions (trade, stash looting, supply trader visits) each have their own toggles and tunable values: chances, cooldowns, thresholds, quantities, rate limits, and budgets.
Log level goes from silent to full tracing with pathing, performance timing, and PDA map markers.

Two reset buttons live under Development. Reset ALL to Defaults restores every setting to factory state and takes effect on the first click. Reset ALL to Stateless Defaults restores factory state and then disables trade, item consumption, stash fill, stash loot, and smart ownership. AlifePlus dispatch and travel still run. Use the stateless preset on installs where another mod owns those systems.

Presets:
  Calm Zone: A-Life Rate 4, Cause Budget 5, Consequence Budget 1, Global Rate Limit 2
  Vanilla+: A-Life Rate 5, Cause Budget 20, Consequence Budget 2, Global Rate Limit 5 (defaults)
  Chaotic: A-Life Rate 20+, Cause Budget 30+, Consequence Budget 5+, Global Rate Limit 15+

---

FAQ:

Do I need modded exes?
  Yes. AlifePlus needs themrdemonized modded exes (2025.9.10 or newer) or AOEngine (v0.55 or newer). Vanilla Anomaly does not expose the APIs the framework relies on.

Does it work with other A-Life mods?
  AlifePlus has no known incompatibilities with warfare or AI addons.
  It will conflict behavior-wise with mods that script squads heavily.

Do I need offline combat enabled?
  No. The engine setting (alife/general/offline_combat: full / on_smarts_only / off; default full) is independent of AlifePlus, which generates its events either way.
  Leave it at the default (full): that is what drives offscreen faction attrition, which most modpacks rely on. Off only freezes that attrition and neither helps nor hurts the framework.
  Reducing or disabling it produces fewer combat events, but the system keeps generating other events, so A-Life activity continues.

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
Muratovnik - fixes and bug reports

---

Usage and License:
- Modpacks: allowed and encouraged. Keep the readme and license files.
- Addons, patches, integrations: allowed. Credit "AlifePlus by Damian Sirbu" visibly on your mod page.
- Reproducing the implementation in other software: not allowed, even with credit.
- Full license in LICENSE file and on GitHub.

Reporting issues and suggestions
Open a bug report or a suggestion at https://github.com/damiansirbu-stalker/AlifePlus/issues/new/choose.
Also discussed on the GAMMA, EFP, Anomaly, and Zona Discord servers.

Before posting, read this readme and the MCM options.

Include:
- Exact steps to reproduce, from a new game or a named save, with expected and actual result.
- xray.log and the mod debug log (MCM log level DEBUG), plus engine build, modlist, load order.
- Describe the behavior. With hundreds of mods and overrides, only the log shows whether this mod was involved and what caused it.
