# Manifesto

AlifePlus finishes the A-Life GSC designed and cut.

The original A-Life was an event-driven world where stalkers lived their own lives, present and acting whether or not the player was watching [10].
They found artefacts, traded with dealers, answered distress calls, and chased their own goals with no regard for the player.
GSC's design documents lay out the whole loop.
A stalker spawns and picks a task by distance and gear.
He walks to it, weighing the dangers on the way.
He eats and heals from his pack, then returns to a trader to turn loot into supplies [8].
The AI grew so capable that stalkers beat the player in firefights and finished quests ahead of him, so GSC reined it in to leave the player room to win [9].
Most of it was gone before Shadow of Chernobyl shipped.
Only smart terrains and campfires survived.

AlifePlus reconnects what GSC left disconnected.
The world reports what happens, whether a death, a healing, an artefact lifted, or a patch of scanned territory.
Then the world answers in kind.
Squads chase down the killers.
Scavengers converge on a massacre.
Factions claim the empty ground.
Faction identity and personality gate every decision, the same five-axis model GSC built [8].
The callbacks, the movement, the smart terrains, and the squads are the engine's own.
Only the wiring between them shipped incomplete.

The mod invents nothing and fakes nothing.

---

## Principles

### Event-Driven Contract

AlifePlus runs on no scripted timer and no hidden director.
Every reaction traces to something that actually happened in the Zone.
A stalker died, a wound was dressed, an artefact was lifted, a squad walked into a camp.
Without an event there is no reaction.
The Zone stays quiet until it does not.

GSC designed the same contract for distress signals.
An event fires, the character's personality weighs it, and one of three outcomes follows.

> "Based on distance, potential gain, and the character's personality, a course of action is selected: (a) Ignore the signal. (b) Kill and loot the sender. (c) Rescue the sender."
> [8]

---

### Physical Simulation Guarantee

AlifePlus spawns no one into a role.
Every stalker and every mutant in a consequence was already out there living its own life before the event pulled it in.
This holds as an invariant.

A squad sent to a massacre site walks there, across the map, past everyone on the way.
A squad fleeing a lost base retreats to the nearest friendly ground on foot.
The travel is real, so it is visible.
It can be interrupted or ambushed.
It can also reach the site too late.

GSC built this movement to carry real weight.
Their design sizes a stalker's food by the length of the walk, his ammunition by the odds of a fight, his armor by the anomalies on the route.

> "He is one of many, whose lives crossed the Zone."
> Alexey Sytianov, Shpil Magazine [2]

---

### Symmetric Entity Treatment

The simulation does not revolve around the player.
Causes fire for every creature.
Consequences resolve for every faction.
A massacre in a corner of the map you will never visit still draws scavengers, and still brings the dead man's faction looking.

The simulation runs on every map at once, including the many you never visit.
Thirty levels live in parallel, and the events off your map far outnumber the ones on it.
A slider lets you bias how much of the mod's attention falls on your level against the rest.

Some characters stand outside it.
Traders, quest givers, companions, mechanics, barmen, faction leaders, guides, and story characters are fixed points.
The world is built on them, and the simulation leaves them alone.
Everyone else is a participant.
Quest targets get no protection.
If your mark gets into a fight and dies before you reach him, you fail the quest.
To protect him would mean faking it.

GSC drew the same line.
A faction leader sits in a guarded hideout as a permanent merchant.
His guards, however many and however armed, live and die like anyone else.

> "Several well-equipped Stalker guards are present in the hideout alongside the leader (the number of guards and their specific gear depend on the faction)."
> [8]

GSC built it this way on purpose.

> "The paradigm of Stalker at the time was that we don't really care about the player. We have this living world."
> Andrii Verpakhovskyi, original game designer [9]

You are not special.

---

### Emergent Narration

If the gameplay is emergent, the telling of it should be emergent too.
No line describing what happened in the Zone is written in advance.

Every PDA message is built the moment it is sent.
Small primitive pieces combine by rule, among them openers, verbs, actor forms, place names, and connectives.
They fill with the live facts of the event, which factions, where, what hour, whose squad.
Roughly two hundred fragments compose into millions of distinct sentences.
The same firefight reads differently on a different day.
The templates are localized, so the Zone speaks each player's language.

GSC designed news exactly this way.
A stalker who witnessed a kill carried it to a trader, who broadcast it to everyone nearby.
The witness chain carried the news.
If the witness died first, the news never happened.

> "To notify player about the offline activity, we used news, which were generated if some character could see an event or its consequences."
> Dmitriy Iassenev, Game Developer (2008) [1]

A fixed phrasebook would lock the Zone to a handful of lines.
Templates filled from live events keep finding new words for things that were already different every time.

---

## Mechanics

Every mechanic works the same way.
The world reports a real event, identity and personality weigh it, and the answer is carried out by stalkers and mutants already living in the simulation.
The decision follows from who you are and what happened.

### Massacre: Scavenge, Investigate, Loot

When bodies pile up in one place, the Zone notices.
Scavenger mutants drift toward the kill to feed.
The dead men's faction sends a squad to find out what happened.
Outlaws hear of the bodies and come to strip them.

The world already drags single corpses.
A mutant finds a body, approaches, and feeds.
A massacre is that instinct at scale, enough death in one spot to pull packs across the map and put a faction on alert.
Kills already shift faction standing under the surface, and investigation is the behavior that finally acts on the shift.

### Squad Kill: Revenge

When the last member of a squad falls, same-faction squads come after the killer.

GSC designed the full consequence chain.
A dying stalker who saw his attacker sent the name to a trader.
The trader weighed reputation and loyalty and could put a bounty on the killer.
Stalkers who later spotted him reported his position, and the hunt began.

> "The Trader issues bounties on Killer Stalkers."
> "When a Stalker spots a Killer, they report the Killer's location to the Traders,"
> "who then broadcast this information to all other Stalkers."
> "Consequently, all Stalkers in the immediate vicinity begin hunting down the Killer."
> [8]

### Base Kill: Support, Flee

When a faction base takes enough losses, two things happen.
Friendly squads nearby converge to reinforce.
The squads caught at the base evacuate to another base of their own faction.

Clear Sky's faction war routed reinforcements to contested points the same way.
A base was worth defending because it anchored the faction's trade and command.

> "The in-game Zone has been shaken by furious war of factions for the new territories, resources, and control points of the important paths to the center of the Zone."
> Clear Sky official gameplay page [4]

### Alpha: Promote

A mutant that keeps killing and surviving climbs ten tiers into an earned individual.
It hits harder and takes more.
It never panics from a wound, and it carries trophies on its corpse.
Its death brings hunters of its own species after the killer.

In vanilla a mutant's combat power is fixed at birth by its species.
A dog that killed fifty stalkers fights exactly like one that never fought.
GSC meant to do otherwise.
Their creature tables rate every species for attack, defense, and courage.
The hierarchy still stayed static by species and never reached the individual.
AlifePlus adds the missing axis.
The dog that keeps winning becomes the dominant dog of the Zone.

The combat identity is the design.
An earlier release carried these effects, then dropped them under outside pressure and ran alphas as tracking-only for a time.
The effects are back in the mod now.
An alpha is felt across a whole fight and never as a single devastating hit.

### Alpha Kill: Targeted

When an alpha dies, mutants of its own species hunt the killer.
To kill the dominant individual of a species costs the killer, because its own kind come after him.

### Wounded: Hunt, Help

When a stalker or the player uses a healing item, two sides move.
Predators and aberrant mutants chase the weakened target.
Same-faction squads rush to help.

The engine models predators in detail, the ambush, the pursuit, the approach, and the feeding.
The hunger drive meant to trigger it was planned and abandoned.
AlifePlus extends the predator's target from dead prey to weakened prey.
GSC's distress logic ran from rescue with food and medicine all the way to killing and looting the caller.
Help and hunt are the two ends of that same signal.

> "A complex AI system with a life simulation system that enables any monster and NPC to act independently on any game level,"
> "with creatures having a full life cycle where they hunt, feed, take rest, and sleep."
> Oles Shishkovtsov, ModDB interview [3]

### Harvest: Rob, Haunt

When a stalker or the player picks up an artefact, the Zone answers in two ways.
Outlaws nearby chase the carrier to take it.
Aberrant mutants feel the artefact's pull and converge on the spot.

GSC designed a full artefact economy.
Dealers generated quests from a map of anomalous activity, and stalkers competed for the same finds.
NPCs already gather artefacts in shipped code.
What was never added is the competitive answer, someone else wanted that artefact too.

> "In early versions of the simulator, a character knew one or several dealers, which had sets of quests"
> "that they generated based on the map of anomalous activity and requests from organizations,"
> "which wanted to get rich from artifacts found in the Zone."
> Dmitriy Iassenev, Game Developer (2008) [1]

### Stash: Loot, Ambush, Fill

A squad arriving at a smart terrain spots a stash within range and acts on it.
It can walk there and take the contents.
It can camp there and ambush whoever comes for it.
It can hide its own surplus inside instead.

The engine indexes every stash and fills half of them with loot.
NPC stash use in the base game stays narrative only.
A surrendering stalker reveals a location, a task reward marks one on the map.
The infrastructure was built and left disconnected.
A stash at a contested supply point is exactly what factions would fight over.

### Area: Conquer, Swarm, Decay

When a squad finds an unguarded smart terrain nearby that is not a base, it walks there and claims it.
A stalker faction conquers a smart and holds it.
A mutant species swarms one, flooding it with its own kind.

The claim takes the ground for itself.
The conqueror's own spawns replace the smart's until the claim decays.
A lair taken by Duty fills with Duty and stops breeding its mutants.
Hold a place long enough and it grows a post of its own.
A trader, barman, mechanic, or medic of the conquering faction mans the ground, with guards around him, stocked below the hand-placed hubs so the hubs stay worth the walk.
A claim decays after a few days.
The post dissolves, the guards stay on as ordinary residents, and the terrain reverts.
Factions expand, fade, and give way to whoever moves in next.

The engine's smart terrain carries no faction field of its own.
All territory control lives in script, and the base game never has a squad decide to take ground on its own.
GSC designed expansion as a staged process, more controlled points, better stalkers fielded, shifting targeting as a faction grows.

> "A control point leading to a place with artifacts is controlled by bandits. Their destruction means that the path to resources is safe."
> Clear Sky pre-release interview [5]

### Infestation: Nest

When a mutant squad's territorial instinct fires, it walks to a smart terrain that suits its species and claims it as a nest.
The original spawns give way to the infesting species.
A per-level cap keeps any one species from swallowing a whole map.

Feral mutants nest in lairs, predators in lairs or buildings, aberrant species in buildings and underground.
The engine already gives every monster a home it rests, retreats, and fights near.
Infestation lifts that from the single creature to the territory.
GSC assigned real breeding rates to the pack species, the rats and rat wolves and dogs, and none to the rest.
Territory that mutants hold is territory where mutants breed.

### Needs: Drives

When a squad reaches a smart terrain, it weighs its unmet needs and pursues the one that hurts most.
A need grows nonlinearly with neglect, so a mild one stays noise and a critical one dominates.
This follows Hull's drive theory and Maslow's hierarchy, the behavioral models GSC worked from when it designed A-Life.

A stalker answers hunger, sleep, and rest at a campfire.
He heals or shelters at a friendly base.
He chases money through anomaly fields and mutant lairs, and restocks at a trader.
He takes a job that fits his faction, a guard post, a scouting run, or anomaly research.
Monolith and Greh worship, while Military and Duty drill.
He seeks company at a fire or back home.
The need is satisfied by the arrival itself.
The stalker walks to the place that matches the need, and the engine plays the eating, healing, or resting it already knows.
AlifePlus destroys nothing and chooses the destination from the need.

The building blocks were all shipped and never connected.
Hunger was planned and left unfinished.
The eating and sleeping states exist, and nothing triggers them by need.
Campfire life, job assignment, and NPC healing all work on their own.
The missing step turned a need into a destination.

> "It was planned that character behaviour would diversify due to introduction of requirements of food and sleep."
> Dmitriy Iassenev, Game Developer (2008) [1]

### Instincts: Scatter, Feed, Sleep, Explore, Socialize

Mutants carry drives the same way stalkers carry needs, and the strongest unmet one moves the pack.
Each instinct is gated by the species' nature.

Scatter pulls a weaker mutant to safety when a stronger species comes within range.
Feed puts predators in reach of prey through shared ground, and the engine handles the kill and the meal when they meet.
Sleep returns a pack to a resting place that fits it, an open field, a lair, a building, or deep underground.
Explore moves the restless species to new territory, while the lair-bound ones stay put.
Socialize draws pack and herd animals toward their own kind, while the solitary hunters keep to themselves.

GSC gave each creature a behavioral profile the engine later flattened into six time-of-day factions.
A dog and a lurker share one engine faction and behave nothing alike.
One runs in a loyal pack, the other hunts alone from cover.
AlifePlus restores the per-species identity GSC documented and the engine erased.

### Day/Night Cycle

Stalkers and mutants keep a daily rhythm.
In their active hours they feed, explore, work, and gather.
In their dormant hours they seek shelter and sleep.

Stalkers are diurnal, and most mutants are too.
The night hunters reverse it.
Bloodsuckers, lurkers, chimeras, zombies, and fractures shelter by day and emerge after dark.
The engine already splits several mutant factions into day and night variants, and AlifePlus carries the split across every drive, for stalkers and mutants alike.

> "Sees better at night and in darkness. At night goes to open areas to hunt."
> monstry.doc, bloodsucker entry [8]

### Cross-Map Travel

Stalkers walk between maps, to resupply, to scout, to visit another base.
The walk belonged to the original lifecycle, a routine part of a stalker's week.
A stalker crosses from one level to the next and finds artefacts.
He returns to a dealer and trades.
Then he takes a new task and heads out again.

The engine ships the walk.
A squad can target another level and cross it on foot, level boundary by level boundary.
Vanilla makes far smarts unattractive, so the simulation almost never sends anyone off-map on its own.
AlifePlus dispatches the squads anyway.
A cause whose destination works at level distance can send a squad across the Zone.

Reach is earned through progress.
A fresh game keeps stalkers home.
The late-game story gates that open the deeper Zone to the player, X-16 and the Brain Scorcher, open it to the squads the same way, and a master-rank commander earns one hop beyond that.
Each trip is a round trip.
The squad walks out, does its business, holds a while, and walks home to where the simulation first saw it.
Off-map traffic is paced so it never snowballs, and a squad already walking cannot be sent again until it returns.

> "Too bad this did not make it into the original game."
> Dmitriy Iassenev, Game Developer (2008) [1]

---

## Emergent Economy

Goods circulate through the Zone, and the circle closes on its own.

A stalker kills and loots the body.
The loot is his, because a kill belongs to the killer, whoever did it, the player or an NPC or a mutant.
The same rule binds every side.
If you drop a man, his gear is yours to claim before an NPC nearby takes it.
If an NPC drops him, the gear is that NPC's, and reaching across the fight to grab it is theft.

He carries the surplus to a trader and sells it.
What he sells surfaces on that faction's shelf.
The merchandise is a real consequence of a real sale, each item traced to a seller.
Another stalker, or the player, buys it, and it leaves the shelf.
The goods moved from a corpse to a trader to a new owner with nothing spawned and nothing destroyed.

No one conjured the goods onto the shelf.
Every item came in on the back of a stalker who found it, and every sale leaves on the back of one who wanted it.
The same artefact moves from a finder to a trader to a buyer.
The world spawns nothing and deletes nothing.

Anomaly already ships an NPC trade system, Alundaio's work and Tronex's upkeep, that almost never fires on its own.
AlifePlus drives it directly, so the shelves move.

Goods also move without money.
Stalkers of one faction barter on the trail, one man's surplus for another's need.
A better gun passes for the worth of the old one, and no money changes hands.

GSC designed the artefact economy as an exchange.
A stalker returning to a dealer traded loot for supplies, and free gear from a merchant is a spawn.
The economy is the lifecycle Iassenev described, running end to end.

---

## Alignment

GSC classified characters along a moral axis and gave each type its own rules.
Nine types crossed principled, self-serving, or unprincipled with good, cold-blooded, or evil.
None of it reached the game.

A Principled Good character behaves one way.

> "Deliberately eliminates humanoid mutants. Would never commit an act of betrayal. Does not loot abandoned equipment."
> [8]

A Neutral Evil character behaves another way entirely.

> "If profitable, will kill and loot those in distress. If profitable, uses PDA signals to lure others into a trap. If the gain is substantial, may betray friends."
> [8]

Three types, three different answers to the same event.
AlifePlus maps that axis onto factions as a hard filter.
Principled factions follow orders and hold ground, among them Duty, Military, Monolith, and ISG.
Self-serving factions act on their own goals, among them Loners, Clear Sky, Ecologists, Freedom, and Mercs.
Outlaw factions operate outside any code, among them Bandits, Renegades, and Sin.

Mutants get the same treatment at the species level.
It restores the identity GSC wrote and the engine collapsed into six time-of-day factions.
GSC sorted creatures by nature and documented a fear hierarchy between the groups.
The engine gave them all one stomach and one faction each.

Cowardly species flee danger and scavenge what others leave.
Feral species run in packs and avenge their fallen.
Predators hunt alone and run down wounded prey.
Aberrant species work through psychic power and defend their lairs.
These four categories stack into a food chain.
The cowardly scatter from everything above them, the feral from predators and aberrant, predators from aberrant.
The aberrant species fear nothing at all.

Alignment sets the wall around what a faction or species can do at all.
Military never flees a base attack.
Ecologists never conquer any territory.
Renegades never investigate a massacre.
Outlaws never help the wounded.
Cowardly mutants never avenge or hunt.
Feral mutants never ambush a stash.
These are structural limits, decided before personality ever rolls.

---

## Personality

GSC modeled personality with five attribute axes that fed probability tables.
The axes were aggressiveness, greed, intelligence, eye range, and relation.
The decision followed from who a character was, so the same event met a different answer from a different character.

AlifePlus runs personality as a probability layer after alignment.
Each faction and each mutant species carries a trait set traced to GSC's own variables, with territory and survival added to round it out.
Alignment decides what is possible, and personality decides how often it happens.
The trait the event touches weights the roll.
A floor and a ceiling hold it in.
Even an unlikely faction acts sometimes, and even an eager one fails sometimes.

Traits key on faction for stalkers and on species for mutants.
This is why a dog and a lurker in the same engine faction behave nothing alike.
The dog has pack loyalty and measured aggression, the lurker is solitary and aggressive.

Aggression drives revenge, the hunt, and the grab for territory.
Greed drives looting, hoarding, and the chase for money.
Perception drives investigation and the search for prey.
Territory drives conquest, shelter, and holding a post.
Relation drives loyalty, the revenge and reinforcement and help and company.
Discipline holds a squad in line.
Its absence breaks that line and sends a squad running.

---

## Range: Awareness and Reach

GSC's eye-range attribute set how far a stalker perceived the world and fed straight into detection.
A sharp-eyed stalker caught enemies others missed.
The number was never shown to the player, and it made each entity's awareness its own.

AlifePlus splits that into three senses that decide how far a squad reaches for a target.
Sight is what a squad sees from where it stands, a stash, an unclaimed outpost, prey at a watering hole.
Radio is what a stalker hears from far off, a massacre, a base under attack, where a trader sits.
Scent is what a mutant smells on the wind, corpses, a blood trail, wounded prey.

Each system maps to a sense.
A squad acts on what it sees, responds to what it hears, and hunts what it smells.
GSC built awareness the same way.
Eye range drew the visible world, and faction distress signals carried word past the edge of it.

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
  The set holds three archives.
  Ai.rar holds the AI design, monster state machines, and combat tables.
  Help.rar holds the A-Life help, faction settings, and lookup tables.
  Tools.rar holds the offline simulation spreadsheets and trader design.
  The monstry.doc species notes are part of this set.
  Leaked in the "Koan's Gift" package, published on the Gameru.net (GAMEINATOR) forum, and catalogued on the STALKER wiki.
- [9] Andrii Verpakhovskyi (original S.T.A.L.K.E.R. game designer), Eurogamer, reported by VideoGamer
  https://www.videogamer.com/news/original-stalker-a-life-so-advanced-that-players-were-constantly-outsmarted-by-ai/
- [10] Dmitriy Iassenev, "A-Life, Emergent AI and S.T.A.L.K.E.R.", AIGameDev.com (2008)
  https://janjilecek.medium.com/a-life-emergent-ai-and-s-t-a-l-k-e-r-70a9cdde3fac
