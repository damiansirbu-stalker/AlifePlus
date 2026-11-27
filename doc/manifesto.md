# Manifesto

AlifePlus builds the A-Life GSC designed and cut.

GSC's design documents lay out an autonomous Zone.
A stalker picks a task by its distance and his gear, walks to it, eats from his pack, heals, and goes back to a trader to turn loot into supplies.
A death spreads as news, a distress signal draws rescue or robbery, and factions hold their ground.
Only part of it shipped. The rest sits in the engine as stubs and commented-out code, and in the design archive as notes nobody finished.

Anomaly fills the gap with a weighted lottery.
Every squad of a faction scores the map the same way, and a coin flip picks its target, which leaves no squad a decision of its own making.
AlifePlus replaces the lottery with causes. An event in the Zone becomes a cause, the cause draws a consequence, and the consequence is the next event.

Each system below names the GSC design or the unfinished engine code it builds on.
Where AlifePlus goes past GSC's design, the text says so, and nothing in the Zone is faked.

---

## Principles

### Reactive to the core

Every decision AlifePlus makes answers an event.
Where X-Ray raises one, such as a death, a healing or an artefact taken, AlifePlus listens to it.
Where X-Ray raises none, AlifePlus adds the event itself.
ap_squad_on_change fires when a squad moves, comes online, changes smart terrain or stops, and a quiet squad gets the same event on its turn. ap_npc_medkit_use fires when an NPC uses a medkit.
There is no director and no script of what should happen. What happens next comes from what happened before.
GSC designed the distress signal the same way, an event weighed by distance, gain and character.

> "Depending on distance, gain and character, a choice of actions: (a) Ignore. (b) Kill and rob. (c) Rescue."
> GSC AI design documents [8]

The engine even carries the hook for it, an encounter check between A-Life objects that GSC commented out with the note "Do not forget to uncomment here!!!".
Other mods hear the same events. ap_squad_on_change is a public callback, and the activity record tells any integrator what AlifePlus does with a squad.

### Physical simulation

A consequence moves a squad that already exists.
A squad sent to a massacre walks there across the map, visible the whole way, and it can be ambushed or arrive too late.
GSC built the walk to carry weight. A stalker's food was sized by the distance to his task, and his ammunition and medkits by the odds of the monsters on the way [8].

> "In Stalker the player is not the main one and not the strongest. He is only one of the people fate brought into the Zone."
> Alexey Sytianov, lead game designer, Shpil (2012) [2]

### Engine first

The callbacks, the movement, the smart terrains and the squads are X-Ray's.
AlifePlus wires them together and never rebuilds in script what the engine already does.
A routed squad gets one field set, its scripted target, and the engine's squad update walks it there.
AlifePlus also repairs a vanilla bug. A squad the game re-homes by script never updates the smart terrain's head count.
AlifePlus corrects that count after every re-home, and on a modpack that already keeps it right, the count stays as it is.

### One of many

The simulation does not revolve around the player. Causes fire for every creature, and consequences resolve for every faction, on every map.
Traders, quest givers and faction leaders stand outside it, the fixed points the world is built on. GSC marked them the same way: "Possibility of killing the trader, not yet" [8].
Story characters, task givers, quest targets, companions and the squads another mod owns are never routed.

> "The paradigm of Stalker at the time was that we don't really care about the player. We have this living world."
> Andrii Verpakhovskyi, original game designer [9]

### News from the event

AlifePlus writes no news in advance.
Each PDA line fills a template for its consequence with the live facts of the event: who, what, where and how long ago.
GSC tied news to a witness. A stalker who saw a murder carried it as news, and if he died before he reached a trader, no one ever learned of it [8].

> "To notify player about the offline activity, we used news, which were generated if some character could see an event or its consequences."
> Dmitriy Iassenev, Game Developer (2008) [1]

---

## The decision model

Every system runs through the same two gates GSC designed: what a faction is allowed to do, and how likely it is to do it.

### Alignment

Alignment decides what a faction or species can do at all.
GSC sorted characters on 2 axes. The first runs from principled through independent to unprincipled, and the second from good to evil. Each type carries rules of its own.
AlifePlus carries the first axis onto factions.
A principled faction, such as the military or Duty, never flees a base.
Ecologists leave every smart terrain to others, and outlaws pass the wounded by.
Mutants take the same treatment by species, grounded in GSC's four creature groups and the fear between them.
The humanoid mutants avoid the bloodsucker, the pseudogiant and the poltergeist.

> "Principled Evil: Will kill and rob those in trouble, and will deliberately set out toward a distress signal to do so. Revenge overrides all tasks."
> GSC AI design documents [8]

### Personality

Alignment decides what is possible, and personality decides how often it happens.
GSC built an Expediency function that turned a character's greed, aggression and the target's worth into an action probability.
AlifePlus runs the same layer. It rolls a faction or species against the traits the event touches, held between a floor and a ceiling, and even an unlikely faction acts sometimes.
This is why a dog and a lurker behave nothing alike. The dog is loyal to its pack, but the lurker hunts alone.

### Range

Awareness has tiers: what a squad sees, and what reaches it by signal.
GSC's eye range set how far a character perceived a threat, and the SOS and PDA signals carried word past the edge of sight.
AlifePlus splits this into sight and radio, and adds a third tier that goes past GSC's design, scent. A pack tracks a blood trail on the wind.

---

## Systems

Every system has an MCM page, in the MCM's order. Each names the GSC design or the unfinished engine code behind it.

### Framework

Framework sets how much of the Zone acts at once.
The A-Life rate sets how many squads get a decision each minute, and the A-Life ratio splits that rate between your map and the rest of the Zone.
A budget per cause family keeps one busy map from flooding the rest, and a cap per causer keeps one killer from drawing every reaction.

### World

#### Off-map

Stalkers walk to a neighboring map. A supply run heads to a faction base, a barter run to an occupied field camp, and a social visit to a campfire.
GSC described the walk as part of the lifecycle, a stalker crossing from one map to another to find artefacts and trade them at a dealer.
The engine ships the walk but scores far smart terrains low, and the simulation almost never sends anyone off-map on its own.
AlifePlus sends them, gated by story progress.
If you open X-16 or the Brain Scorcher, the deeper Zone opens to the squads as well, and a veteran commander earns one more map.

> "I followed a stalker from one level to another, saw how he searched for artifacts, found them, returned to the dealer, approached him, traded, picked a new quest, went on."
> Dmitriy Iassenev, Game Developer (2008) [1]

### Territory

#### Conquest

A squad finds an empty smart terrain that is not a base, walks there, and takes it for its faction.
The conqueror's spawns replace the smart terrain's until the claim decays, and wiping the holders does not hand the ground over.
A weak faction on the level conquers more and a dominant one less, each up to a set number of areas.
A held area grows an outpost: a trader, barman, mechanic or medic of the winning faction, and a guard detail. Its stock sits below the hand-placed hubs, which stay worth the walk.
Each service stocks a shelf for his role, a barman a pantry and a medic a dispensary.
He wears his faction's look, carries a sidearm and shows on your PDA, and trader stock addons treat him like any trader.
The engine already carries the field for it. A smart terrain's faction and faction control drive which faction respawns there, and vanilla uses them at only a few places.
GSC listed the leader of a stalker faction as one of its 3 kinds of trader, beside the ordinary trader and the scientist [8].

#### Swarm

A mutant species takes an empty smart terrain the way a faction conquers one, and it respawns only that species while held.
Only pack and lair species take ground. Solo bosses and vermin never do, and mutants swarm more where they are few.
GSC gave only a few creature types a real breeding rate and left every other at zero [8].

#### Infest

A pack with an alpha turns a den into a nest. The nest holds more squads than a swarm and replaces the smart terrain's spawns until it decays.
A cap per level keeps the nests from taking over a whole map.
The engine already gives every monster a home it rests, retreats and fights near. Infestation lifts that home from one creature to a territory.

### Creatures

#### Alpha

A mutant that kills and survives climbs ten levels into an earned individual.
It hits harder and takes less damage, shrugs off panic, and carries species trophies on its corpse.
A dust shimmer on its body marks the pack leader on sight.
GSC rated every creature for how it fights, an attack and defence courage score per species, so the hierarchy was designed but fixed by species.
The individual axis goes past GSC's design. The dog that keeps winning becomes the dominant dog of the Zone.

### Economy

#### Loot claim

A kill belongs to the killer, whoever did it, the player, an NPC or a mutant.
If you drop a man, his body is yours while you have a clear line to it, and an NPC nearby leaves it. If an NPC drops him, the gear is that NPC's squad's while one of them can see the body.

#### Loot policy

A looting stalker keeps a bounded share of what he took from a body, the costliest items first, and drops the rest.
He keeps a spare scope or silencer that fits his gun.

#### Trade

A stalker carries his surplus to a trader and runs a real buy and sell, selling spares and restocking by rank, a few items per visit.
A veteran also buys the cheapest scope, silencer or launcher that fits his gun, the way he buys ammo for it.
GSC designed the trader to price goods by supply and money on hand, and to give out artefact orders.
Anomaly ships the full cycle, Alundaio's work and Tronex's upkeep, but it almost never fires by itself, and AlifePlus drives it directly.

#### Barter

Two stalkers of one faction barter on the trail, one man's surplus for another's need, and a better gun passes to the man who carries a worse one.
Money never changes hands between them.
Ammo and attachments go only to a man whose gun takes them.

#### Market

What a faction's stalkers sell surfaces on that faction's hub shelves, at a premium, for the player to buy.
What they buy thins the same shelves, and a trader runs short of the staples his faction keeps buying.
Goods move from a corpse to a trader to a new owner, and NPC trading finally decides part of what the player finds for sale.

### News

The news surfaces these events to the player. Each line fills a template for its consequence with who, what and where, in the voice of a stalker on the radio.
You also overhear the stalkers near you, when one resupplies at a trader, passes a gun to a squadmate or strips a body.
GSC designed the templates and the witness chain. A stalker had to see a thing to tell it, and with no witness the news never happened.

### Causes

#### Reactions

##### AlphaKill

When an alpha dies, mutants of its species hunt the killer.

##### Massacre

When bodies pile up in one place, scavenging mutants drift toward the kill to feed, the dead men's faction sends a squad to find out what happened, and outlaws come to strip the bodies.
GSC designed the reaction to a squadmate's corpse.
On a kill the engine shifts the goodwill of the dead man's group and faction toward the killer, and sends no one after him.

##### SquadKill

When the last member of a squad falls, squads of the same faction come after the killer.
A chasing squad homes on its target's live position. It waits while the target shelters in a base or an emission passes, and gives up when the target dies, leaves the map or stops being an enemy.
GSC designed the full chain. A dying stalker named his attacker to a trader, and the trader put a bounty on the killer.

> "The Trader issues bounties on Killer Stalkers. When a Stalker spots a Killer, they report the Killer's location to the Traders, who then broadcast this information to all other Stalkers."
> GSC AI design documents [8]

##### BaseKill

When a faction base takes enough losses, friendly squads converge to reinforce it, and the squads caught at the base evacuate to another base of their faction.
Clear Sky's faction war was fought for territory, resources and the control points on the paths to the center of the Zone [4].

##### Wounded

When a stalker or the player uses a healing item, predators and aberrant mutants close on the weakened target, and squads of the same faction rush to help.
GSC designed both answers as the SOS response, a stalker rescued or finished off on his distress signal.
The engine carries the predator states, with the hunger drive that would trigger them left as a stub.

##### Harvest

When a stalker or the player picks up an artefact, outlaws chase the carrier to rob him, and aberrant mutants converge on the spot.
GSC designed the competition, the trader's artefact orders and the character who sees a rich rival, and the engine carries artefact spawning as commented-out code.
The aberrant draw goes past GSC's design, as a pull of the artefact itself.

#### Opportunities

##### Stash

A squad spots a stash within range and acts on it. It walks there to take the contents, camps to ambush whoever comes for it, or hides its surplus inside.
The engine indexes every stash and fills half with loot.
The ambush and the fill go past GSC's design.

#### Instincts

Mutants carry drives too, and the strongest unmet one moves the pack.
Scatter pulls a weaker species to safety when a stronger one comes near. Feed puts predators in reach of prey.
Slumber returns a pack to a resting place that fits it. Roam moves the restless species, and Pack draws the herd animals together.
Stalkers and mutants both keep a day and night clock. The diurnal species are active by day, and the night hunters after dark.
GSC documented each species' feeding, habitat and fears, and gave every monster a home point the drive works from.
GSC wrote the night hunters out plainly, the bloodsucker that sees better in the dark and goes out into the open to hunt.
The engine already splits several mutant factions into day and night variants.

> "Sees better at night and in darkness. At night goes out into open areas to hunt."
> monstry.doc, bloodsucker entry [8]

#### Needs

When a squad reaches a smart terrain, it weighs its unmet needs and pursues the one that hurts most.
A stalker answers hunger, sleep and rest at a campfire, and heals or shelters at a base.
He chases money through anomaly fields, restocks at a trader or barters with a peer, takes a faction job, and seeks company.
GSC wrote this loop out in full, the offline stalker fixing his state with a medkit, food, antirad or sleep, then going to a profitable trader by distance and price.
The engine carries the eating and sleeping states, and nothing triggers them by need.

> "It was planned that [cut] character behaviour would diversify due to introduction of requirements of food and sleep."
> Dmitriy Iassenev, Game Developer (2008) [1]

### Flavor

#### Dynamic jobs

A calm stalker near you takes a short job where he stands, a smoke, the PDA, a detector sweep at an anomaly or a drill with his faction, then walks on.
GSC listed playing a specific animation among the ways a character reacts to what happens around him.
The engine ships the states in its state library and plays them only where a level designer scripted one, at a single smart terrain slot.
AlifePlus frees them from the slot, and the stalker's situation picks the job.

> "Example reactions: change of relation, play a specific animation, say a line, send a PDA message."
> GSC AI design documents [8]

---

## Sources

- [1] Dmitriy Iassenev (AI programmer, SoC/CS), Game Developer (2008)
  https://www.gamedeveloper.com/game-platforms/interview-inside-the-ai-of-i-s-t-a-l-k-e-r-i-
- [2] Alexey Sytianov (lead game designer), interview in Shpil magazine, issue 4, 2012. Russian original, full text:
  https://true-stalker.at.ua/news/aleksej_sytjanov_intervju_dlja_shpil/2012-05-11-2
  English summary: https://stalker.fandom.com/wiki/Interview_with_Alexey_Sytianov_in_Shpil
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
