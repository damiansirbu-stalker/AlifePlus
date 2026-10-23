# Manifesto

AlifePlus builds the A-Life GSC designed and cut.

GSC's own design documents lay out an autonomous Zone.
A stalker spawns and picks a task by distance and gear.
He walks to it and eats from his pack.
He heals his wounds and returns to a trader to turn loot into supplies.
Every death spreads as news.
Distress draws rescue or robbery.
Factions hold and defend their ground.
Only a fraction of it shipped.
The rest sits in the engine as stubs and commented-out code, and in the design archive as the architect's own notes to finish it.

Anomaly filled the gap with a weighted lottery, the same draw for every squad, so no squad decides for itself.
AlifePlus reconnects what GSC left disconnected.
Engine events become causes, causes become consequences, and each consequence is the cause of the next.

Every system here is a behaviour GSC designed in their documents, or one the engine carries unfinished in its own code.
The manifesto names the source behind each.

The mod invents nothing and fakes nothing.

---

## Principles

### Event-Driven Contract

AlifePlus runs on no scripted timer and no hidden director.
Every reaction traces to something that happened in the Zone, a death, a healing, an artefact lifted, a squad walking into a camp.
A squad also looks around when it arrives at a smart terrain.
It acts on what it finds, a stash, empty ground, or an unmet need.
GSC designed the same contract for distress signals, where an event fires, personality weighs it, and one of three outcomes follows.
The engine even carries the seam for it, an NPC-to-NPC encounter hook GSC stubbed and left with the note "Do not forget to uncomment here."

> "Based on distance, potential gain, and the character's personality, a course of action is selected: (a) Ignore the signal. (b) Kill and loot the sender. (c) Rescue the sender."
> GSC AI design documents [8]

### Physical Simulation Guarantee

AlifePlus spawns no one into a role.
Every stalker and every mutant in a consequence was already out there before the event pulled it in.
A squad sent to a massacre walks there across the map, visible the whole way, and can be ambushed or arrive too late.
GSC built this movement to carry real weight, sizing a stalker's food by the length of the walk and his ammunition by the odds of a fight.

> "He is one of many, whose lives crossed the Zone."
> Alexey Sytianov, Shpil Magazine [2]

### Engine-Native Behaviour

Every behaviour comes from the engine and the Anomaly layer.
The callbacks, the movement, the smart terrains, and the squads are the engine's own.
AlifePlus wires them together and never rebuilds in script what the engine already does.

### Symmetric Entity Treatment

The simulation does not revolve around the player.
Causes fire for every creature and consequences resolve for every faction.
The whole Zone runs at once, on every map, and the events off your level far outnumber the ones on it.
Some characters stand outside it, the traders, quest givers, and faction leaders the world is built on, and GSC marked them the same way, with "killing the trader" left as "not yet."

> "The paradigm of Stalker at the time was that we don't really care about the player. We have this living world."
> Andrii Verpakhovskyi, original game designer [9]

### Emergent Narration

AlifePlus writes no message in advance.
It builds every PDA line when it is sent, from primitive pieces combined by rule and filled with the live facts of the event.
GSC designed news the same way, generated from events a character witnessed, and carried as a pre-mortem signal that named the cause of death.

> "To notify player about the offline activity, we used news, which were generated if some character could see an event or its consequences."
> Dmitriy Iassenev, Game Developer (2008) [1]

---

## The decision model

Every system runs through the same two gates GSC designed, what a faction is allowed to do, and how likely it is to do it.

### Alignment

Alignment decides what a faction or species can do at all.
GSC classified characters on a moral axis, principled or self-serving or unprincipled, crossed with good or cold-blooded or evil, each type with its own rules.
AlifePlus carries the first axis onto factions.
Military never flees a base.
Ecologists never conquer any ground.
Outlaws never help the wounded.
Mutants take the same treatment by species, grounded in GSC's four creature groups and the fear hierarchy between them.
The humanoid mutants avoid the bloodsucker, the pseudogiant, and the poltergeist.

> "Principled Evil: Will kill and rob those in trouble, and will deliberately set out toward a distress signal to do so. Revenge overrides all tasks."
> GSC AI design documents [8]

### Personality

Alignment decides what is possible, and personality decides how often it happens.
GSC built an Expediency function that turned a character's attributes into an action probability, and named intelligence as one of the combat attributes to introduce.
AlifePlus runs the same probability layer, rolling a faction or species against the trait the event touches, held between a floor and a ceiling so even an unlikely faction acts sometimes.
A dog and a lurker share one engine faction and roll nothing alike, the dog with pack loyalty, the lurker solitary and aggressive.

### Range

Awareness has tiers, what a squad sees and what reaches it by signal.
GSC's eye-range attribute set how far a stalker perceived threats, and the SOS and PDA signals carried word past the edge of sight.
AlifePlus splits this into sight and radio, and adds a third tier of our own, scent, so a pack tracks a blood trail on the wind.

---

## Systems

Every system has its own MCM page, in this order. Each names the GSC design or the unfinished engine code behind it.

### World

#### Cross-Map Travel

Stalkers walk between maps to resupply, to scout, to visit another base.
GSC described the walk as part of the lifecycle, a stalker crossing from one map to another to find artefacts and trade them at a dealer.
The engine ships the walk but makes far smarts unattractive, so the simulation almost never sends anyone off-map on its own.
AlifePlus dispatches, gated by story progress, X-16 and the Brain Scorcher opening the deeper Zone to the squads as they open it to the player, and a master-rank commander earning one hop beyond that.

> "I followed a stalker from one level to another, saw how he searched for artifacts, found them, returned to the dealer, approached him, traded, picked a new quest, went on."
> Dmitriy Iassenev, Game Developer (2008) [1]

### Mutators

#### Conquer

A squad finds an unguarded smart terrain that is not a base, walks there, and claims it for its faction.
The claim takes the ground for itself, and the conqueror's own spawns replace the smart's until it decays.
Hold a place long enough and it grows a post, a trader, barman, mechanic, or medic of the winning faction, stocked below the hand-placed hubs so the hubs stay worth the walk.
The engine's smart terrain carries no faction field of its own, so all of this lives in script.
GSC built the faction around exactly this NPC, the leader who runs a guarded hideout as a permanent merchant.

#### Swarm

A mutant species floods a smart terrain with its own kind, the species twin of a faction's conquest.
GSC gave the pack species a territory to patrol and a real breeding rate, the rats, rat wolves, and dogs, while the rest never breed.

#### Infest

A mutant squad claims a smart terrain as a nest, and the original spawns give way to the infesting species, capped so no one species swallows a whole map.
The engine already gives every monster a home it rests, retreats, and fights near, and infestation lifts that from the single creature to the territory.

### Creatures

#### Alpha

A mutant that kills and survives climbs ten tiers into an earned individual.
It hits harder and takes more.
It shrugs off panic and carries species trophies on its corpse.
A dust-shimmer mark on its body reads the pack leader on sight.
GSC rated every creature for how it fights, an Attack and Defend score per species feeding a combat matrix, so the hierarchy was designed but static by species.
The individual axis is ours, the dog that keeps winning becomes the dominant dog of the Zone.

### Economy

#### Loot

A kill belongs to the killer, whoever did it, the player or an NPC or a mutant.
If you drop a man his gear is yours to claim before an NPC nearby takes it, and if an NPC drops him the gear is that NPC's.

#### Trade

A stalker carries his surplus to a trader and runs a real buy and sell, selling spares and restocking by rank.
GSC designed the trader to price goods by supply and money on hand, and to generate artefact orders.
Anomaly ships the full cycle, Alundaio's work and Tronex's upkeep, and it almost never fires on its own, so AlifePlus drives it directly.

#### Barter

Two stalkers of one faction barter on the trail, one man's surplus for another's need, and a better gun passes for the worth of the old one.
Money never changes hands between them.

#### Market

What NPCs sell surfaces on that faction's hub shelves, at a premium, for the player to buy.
The goods move from a corpse to a trader to a new owner.
The world spawns nothing and deletes nothing.

### News

The news layer surfaces these events to the player, composed at runtime from a grammar.
GSC designed the templates and the witness chain, so a stalker had to see a thing to tell it, and with no witness the news never happened.

### Reactions

#### Alpha Kill

When an alpha dies, mutants of its own species hunt the killer.

#### Massacre

When bodies pile up in one place, scavenging mutants drift toward the kill to feed, the dead men's faction sends a squad to find out what happened, and outlaws come to strip the bodies.
GSC designed the reaction to a squadmate's corpse.
The engine already tracks the kill-attitude shift between factions and acts on none of it.

#### Squad Kill

When the last member of a squad falls, same-faction squads come after the killer.
GSC designed the full chain, the dying stalker naming his attacker to a trader, who put a bounty on the killer.

> "The Trader issues bounties on Killer Stalkers. When a Stalker spots a Killer, they report the Killer's location to the Traders, who then broadcast this information to all other Stalkers."
> GSC AI design documents [8]

#### Base Kill

When a faction base takes enough losses, friendly squads converge to reinforce and the squads caught at the base evacuate to another base of their own faction.
GSC built the base around a guarded hideout worth defending, and Clear Sky fought its faction war over exactly these control points.

#### Wounded

When a stalker or the player uses a healing item, predators and aberrant mutants close on the weakened target, and same-faction squads rush to help.
GSC designed both answers as the SOS response, a stalker rescued or finished off on his distress signal.
The engine carries the predator states, with the hunger drive that would trigger them left as a stub.

#### Harvest

When a stalker or the player picks up an artefact, outlaws chase the carrier to rob it, and aberrant mutants converge on the spot.
GSC designed the competition, the trader's artefact orders and the character who sees a rich rival, and the engine carries artefact spawning as commented-out code.
The aberrant draw is our own extension of the artefact's pull.

### Opportunities

#### Stash

A squad spots a stash within range and acts on it, walking there to take the contents, camping to ambush whoever comes for it, or hiding its own surplus inside.
The engine indexes every stash and fills half with loot.
It leaves the handler for NPC stash use empty.
The ambush and the fill are ours.

### Instincts

Mutants carry drives the same way, and the strongest unmet one moves the pack.
Scatter pulls a weaker species to safety when a stronger one comes near.
Feed puts predators in reach of prey.
Slumber returns a pack to a resting place that fits it.
Roam moves the restless species, and Pack draws the herd animals together.
Stalkers and mutants both keep a day and night clock, the diurnal active by day and the night hunters after dark.
GSC documented each species' feeding, habitat, and the fear hierarchy, and gave every monster a home point the drive works from.
GSC wrote the night hunters explicitly, the bloodsucker that sees better in the dark and goes out into the open to hunt.
The engine already splits several mutant factions into day and night variants.

> "Sees better at night and in darkness. At night goes out into open areas to hunt."
> monstry.doc, bloodsucker entry [8]

### Needs

When a squad reaches a smart terrain, it weighs its unmet needs and pursues the one that hurts most.
A stalker answers hunger, sleep, and rest at a campfire, heals or shelters at a base, chases money through anomaly fields, restocks at a trader, takes a faction job, and seeks company.
GSC wrote this loop out in full, the offline stalker fixing his state with a medkit, food, antirad, or sleep, then going to a profitable trader by distance and price.
The engine carries the eating and sleeping states, and nothing triggers them by need.

> "It was planned that character behaviour would diversify due to introduction of requirements of food and sleep."
> Dmitriy Iassenev, Game Developer (2008) [1]

---

## Sources

- [1] Dmitriy Iassenev (AI programmer, SoC/CS), Game Developer (2008)
  https://www.gamedeveloper.com/game-platforms/interview-inside-the-ai-of-i-s-t-a-l-k-e-r-i-
- [2] Alexey Sytianov (game designer, SoC), Shpil Magazine
  https://stalker.fandom.com/wiki/Interview_with_Alexey_Sytianov_in_Shpil
- [3] Oles Shishkovtsov (engine programmer), ModDB
  https://www.moddb.com/features/stalker-interview
- [4] Clear Sky official gameplay page
  https://cs.stalker-game.com/en/?page=gameplay
- [5] Clear Sky pre-release interviews, The Cutting Room Floor
  https://tcrf.net/Prerelease:S.T.A.L.K.E.R.:_Clear_Sky/Interviews
- [8] GSC Game World internal AI design documents, pre-release, circa 2002-2006.
  Ai.doc has the AI design and the nine character types, monstry.doc the creatures, Killing_NPC.doc death and the offline loop, traders.doc the economy.
  The EFC tables hold the combat and probability data: creature_courage, CreatureEffectiveness, Expediency, BirthProbability.
  Leaked in the "Koan's Gift" package, published on the Gameru.net forum, and catalogued on the STALKER wiki.
- [9] Andrii Verpakhovskyi (original S.T.A.L.K.E.R. game designer), Eurogamer, reported by VideoGamer
  https://www.videogamer.com/news/original-stalker-a-life-so-advanced-that-players-were-constantly-outsmarted-by-ai/
- [10] Dmitriy Iassenev, "A-Life, Emergent AI and S.T.A.L.K.E.R.", AIGameDev.com (2008)
  https://janjilecek.medium.com/a-life-emergent-ai-and-s-t-a-l-k-e-r-70a9cdde3fac
