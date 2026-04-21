# Manifesto

AlifePlus completes GSC's original A-Life design using the X-Ray engine architecture it was built on.

The original A-Life was an event-driven simulation where NPCs lived autonomous lives.
They found items, traded with dealers, responded to threats, and pursued goals independently of the player.
GSC's internal design documents describe the full autonomous lifecycle: a stalker spawns, selects a task based on distance and equipment, walks to the destination, evaluates threats en route, heals and eats from inventory, and returns to a trader to exchange loot for supplies [8].
Most of it was cut before Shadow of Chernobyl shipped.
What remained was smart terrains and campfires.

AlifePlus reconnects what GSC left disconnected.
Causes detect events in the world: deaths, healings, artefact pickups, territorial scans.
Consequences react to them: squads pursue killers, scavengers converge on massacre sites, factions claim empty territory.
A personality system gates every decision through faction identity, the same approach GSC designed with their five-axis personality model and the Expediency lookup function [8].
The engine callbacks, the entity movement, the smart terrain system, the squad lifecycle are all original.
The wiring between them is what shipped incomplete.

Nothing is invented and nothing is faked.

---

## Principles

### Event-Driven Contract

Nothing runs unless the engine says something happened.
If nothing happens in the simulation, the mod consumes zero cycles.

A death propagates through four engine layers (`GE_DIE` -> `eDeath` -> `notify_on_member_death` -> `alife().on_death`) before reaching Lua.
A smart terrain transition fires on every graph node the entity crosses (`on_location_change`).
GSC even stubbed an NPC-NPC encounter hook (`alife_monster_abstract.cpp`, `#pragma todo("Do not forget to uncomment here!!!")`), but never connected it.

AlifePlus listens to these callbacks through a producer.
Each one flows through a gate chain, evaluates a predicate, and publishes a cause to the event bus.
Consequences subscribe to causes.
Neither side references the other.

GSC's design documents describe the same pattern for SOS signals: an event fires, personality evaluates, and one of three outcomes dispatches [8].

> "Based on distance, potential gain, and the character's personality, a course of action is selected: (a) Ignore the signal. (b) Kill and loot the sender. (c) Rescue the sender."
> [8]

---

### Physical Simulation Guarantee

Nothing is spawned, teleported, or created to fill a role.
Every entity involved in any consequence was already there, living its own A-Life.
This is an invariant, not a guideline.

AlifePlus consequences use the engine's own movement.
A squad scripted to a massacre site walks there through graph traversal, visible to every entity it passes on the way.
A squad fleeing a base kill retreats to the nearest friendly base through the same precondition routing the engine uses for simulation.

GSC designed this movement to support real logistics.
Their design documents calculate food quantity from travel distance, ammunition from encounter probability, and protective gear from the anomaly map along the route [8].

Every squad selects its own smart terrain target (`alife_monster_brain.cpp`) and walks there through A* graph traversal (`alife_monster_detail_path_manager.cpp`).
Position updates are atomic (`alife_graph_registry_inline.h`).
Squad members sync to their commander (`alife_online_offline_group.cpp`).
Online and offline switching is distance-based with hysteresis (`alife_switch_manager.cpp`).

> "The player is not the most powerful or the strongest. He is one of many, whose lives crossed the Zone."
> Alexey Sytianov, Shpil Magazine [2]

---

### Symmetric Entity Treatment

Causes fire for all creatures equally.
Consequences resolve for all factions equally.
The simulation does not revolve around the player.

AlifePlus protects traders, quest givers, companions, mechanics, barmen, leaders, guides, and story characters through a shared predicate before any squad selection.
Quest targets are not protected.
Your target gets in a fight and dies before you reach him, and you fail the quest.
Protecting quest targets from the simulation would mean faking it.

GSC designed hostile faction SOS responses where a member of the opposing faction could be dispatched to eliminate the caller [8].
The simulation applied the same rules regardless of who sent the signal.
AlifePlus does the same: both stalker and mutant factions enter the same causes, pass through the same gates, and receive the same consequences.

You are not special.

---

### Emergent Narration

If the gameplay is emergent, the narration of it should be emergent too.
Nothing describing what happened in the Zone is pre-written.

AlifePlus events fire from cause-consequence chains: a death, a massacre, a territory claim, a pack scattering.
The news layer that surfaces these events to the player uses the same principle.
No canned sentences.
Every PDA message is composed at runtime from a recursive grammar: small primitive pieces (openers, verbs, actor forms, place references, connectives) combine through rule expansion, with live event data (faction, location, time, commander names) injected per tick.
Roughly 200 text fragments compose into millions of distinct sentences.
Same fight, different day, different words.
Same primitives, different locale.

GSC designed news exactly this way.
An NPC who saw a kill recorded the event, passed it to a trader, who broadcast to all nearby stalkers [8].
The witness chain carried the news.
If the witness died before sending it, the news never happened.

> "To notify player about the offline activity, we used news, which were generated if some character could see an event or its consequences."
> Dmitriy Iassenev, Game Developer (2008) [1]

AlifePlus implements this explicitly.
Consequences record events to a journal (ring buffer, session-only).
The composer reads the journal, detects patterns between recent events (cascade, hot zone, retaliation, chase resolved), and narrates them through the grammar tree.
Event-connected stalkers are preferred as senders: a same-faction witness first, a friendly stalker second.
When no witness is available, the tick is wasted and no message goes out.

The grammar itself is localized.
English and Russian ship together with identical ID coverage.
Russian is natively authored, not translated word for word.
Each locale provides its own branches through the same rule symbols, so a new language is an XML file, not a code change.

> "A discovered character corpse... Pre-mortem PDA Signal (sent to all known Traders; conveys the cause of death: perishing in an anomaly, death by monsters, death by Stalker weaponry, death due to a status effect, or death by a physical object)."
> [8]

Pre-written text locks the simulation to a finite phrasebook.
Grammar over live data lets the Zone keep finding new words for things that are already different every time.

---

## Architecture

### Event Pipeline

AlifePlus splits event processing into two stages: a producer and a consumer.
The producer receives engine callbacks, filters them through a chain of gates, evaluates predicates, and publishes causes to an event bus.
The consumer receives causes and dispatches them to consequence handlers.
Causes and consequences are fully decoupled.
A cause does not know which consequences subscribe to it.
A consequence does not know which cause published the event.

X-Ray already splits processing the same way.
Current-level entities update every render frame through `update_switch` with a real-time budget (`alife_update_manager.cpp`).
Background entities update through `update_scheduled`, round-robining a fixed count per tick across all offline entities.
GSC's design documents take this further: the offline stalker FSM branches on world events (SOS signals, enemy encounters, task completion), evaluates personality, and selects a response [8].
AlifePlus formalizes this into two pipeline variants.
Reactive events are world-state changes: a death, a healing, an artefact pickup.
Radiant events are ambient observations: a squad looks around and notices a stash, empty territory, or an unmet need.
Both variants share the same gate chain but with different admission rates.

---

### Radiant Evaluation

Squads evaluate their surroundings when they arrive at a smart terrain.
A squad looks around for stashes, unguarded outposts, threats, or unmet needs.
All radiant causes originate here: stash discovery, territory conquest, and the nine stalker drives.

GSC's smart terrain system already defines this moment.
The engine fires `on_location_change` on every graph node transition and gates task selection through `can_choose_alife_tasks` and `smart_terrain_choose_interval`.
GSC's help documentation classifies smart terrains by strategic type: territory for defensible ground, resource for supply points [8].
Their job system assigns roles per time of day: guards and patrols during the day, sleep at night, rangers during weather transitions [8].
The evaluation moment was built into the engine.
What was missing was the decision layer on top of it.

> "I was very happy with the work done when I followed a stalker from one level to another, saw how he searched for artifacts, found them, returned to the dealer, approached him, traded, picked a new quest, went on. Too bad this did not make it into the original game."
> Dmitriy Iassenev, Game Developer (2008) [1]

---

### Synthetic Callbacks

Not every event GSC designed has an engine callback behind it.
When one is missing, AlifePlus wraps the existing function at runtime, calls the original, and emits a new event on top of it.
No base scripts are modified.

The wounded cause works this way.
The engine has no callback for NPC medkit use, so a synthetic hook intercepts the healing handler and broadcasts the event (`AddScriptCallback`, `SendScriptCallback`).
Anomaly itself uses the same mechanism for pseudogigant stomp callbacks, net spawn/destroy lifecycle hooks, and the bullet callback chain.

GSC's design documents describe systems that required callbacks the engine never provided.
The murder witness system needed an NPC-NPC encounter event to spread news [8].
The SOS response system needed a distress signal broadcast [8].
The encounter hook exists in engine source but was never connected (`alife_monster_abstract.cpp`).
Synthetic callbacks fill the same gap.

> "A discovered character corpse... Pre-mortem PDA Signal (sent to all known Traders; conveys the cause of death: perishing in an anomaly, death by monsters, death by Stalker weaponry, death due to a status effect, or death by a physical object)."
> [8]

---

### A-Life Ratio

Thirty maps simulate simultaneously, and the same causes fire and the same consequences resolve on every one of them.
The rules are identical but the frequency is not.

The engine was designed with this differential.
Background levels use a slower movement speed than the current level (`m_fGoingSpeed` vs `m_fCurrentLevelGoingSpeed` in `alife_movement_manager_holder.h`).
Current-level entities update every render frame while background entities round-robin at a fixed count per tick (`alife_update_manager.cpp`).
Background events outnumber current-level events roughly 50:1.
Anomaly removes the movement speed differential by setting both values to 3 in `m_stalker.ltx`, but the update frequency gap remains.

GSC also tracked per-terrain survival risk across all maps.
Their surge death probability table ranges from 1% on sheltered terrain to 85% on open ground [8].
The simulation was always meant to be map-aware.

AlifePlus restores the frequency differential at the cause evaluation layer through a Bresenham ratio gate.
The default yields a 4:1 admission ratio favoring the current level.
A slider from -10 (background only) to +10 (current level only) lets the player tune the balance.

> "The engine defines two movement speeds: `m_fGoingSpeed` for background levels and `m_fCurrentLevelGoingSpeed` for the current level. Current-level entities process every render frame through `update_switch`. Background entities round-robin through `update_scheduled` at a fixed count per tick."
> `alife_movement_manager_holder.h`, `alife_update_manager.cpp` [6]

---

### Protection Gates

Not every entity in the simulation should be a participant.
Traders, quest givers, companions, mechanics, barmen, leaders, and story characters exist as fixed points in the world.
AlifePlus filters them out through a four-layer protection model: at the producer before any cause evaluates, inside reactive cause predicates for the entity being acted on, in consequence squad searches for every candidate responder, and at the squad scripting boundary.

GSC made the same distinction.
Their faction design places the leader as a permanent merchant inside a guarded hideout, with well-equipped stalker guards whose number and gear depend on the faction [8].
If the leader dies, a new one emerges after the next emission from the most skilled remaining member [8].
The leader is a fixture. The guards are simulation participants.

AlifePlus enforces this separation through a shared predicate that covers permanent characters, active roles (task givers, companions), task targets, and squads owned by other mods.
External mods register ownership filters so AlifePlus never scripts a squad another mod controls.

> "Several well-equipped Stalker guards are present in the hideout alongside the leader (the number of guards and their specific gear depend on the faction). If the leader is killed, a new leader emerges within the faction following the next Emission."
> [8]

---

### Rate Limiting

The engine fires `squad_on_update` for every squad every tick.
Online squads fire at frame rate, offline squads fire through the A-Life scheduler round-robin.
The raw volume is thousands of calls per minute, and most of them carry no meaningful event.

GSC faced the same problem and solved it with budgets.
Current-level entities get a dedicated processing budget through `setup_current_level` (`alife_graph_registry.cpp`).
Background entities share a fixed-count round-robin across all offline squads (`update_scheduled` in `alife_update_manager.cpp`).
X-Ray already rate-limits the simulation at the entity processing layer.

AlifePlus rate-limits at the cause evaluation layer through five independent mechanisms.
A pacer rejects calls within a configurable interval of the last accepted call, eliminating over 99% of raw volume with a single timestamp comparison.
A per-cause sliding window prevents any single cause from flooding the pipeline.
A per-consequence token bucket prevents any single consequence from monopolizing responses.
A global consequence counter caps total radiant activity per minute.
Reactive consequences from deaths, wounds, and pickups bypass the global limit entirely because they are rare, high-significance events that should never be starved by ambient activity.

The radiant pipeline is extensible.
New causes register a predicate on the radiant callback pool and the pipeline evaluates them alongside existing ones.
The same mechanism works for other engine events: squad entering or leaving a smart terrain, vertex changes, level transitions.
Any stable heartbeat the engine provides can feed the gate chain.

> "Now, instead of creation of an algorithm for choice of current needs and their satisfaction, a simpler model has been used, which offloaded this work to the smart terrains."
> Dmitriy Iassenev, Game Developer (2008) [1]

---

### Round-Robin Dispatch

When a cause publishes an event, multiple consequences may subscribe to it.
A massacre can trigger both scavengers converging and the victim's faction investigating.
A squad kill can trigger both revenge pursuit and retreat to base.
The dispatch order determines which consequence gets the first chance to claim a squad.

The engine uses priority-based routing for simulation targeting.
Each smart terrain has property weights for base, territory, resource, lair, and surge (`simulation_objects_props.ltx`).
Each faction has behavior coefficients that multiply these weights differently: stalkers weight resource at 2x and surge shelter at 3x, while monsters weight lair at 1x and ignore bases entirely (`simulation_objects.script`).
The final priority factors in distance, and the engine picks from the top 5 candidates.
GSC's faction settings add another layer: per-faction sim_prior tables at each expansion level, so a faction's targeting preferences shift as it grows [8].

This approach serves simulation targeting well, but consequence dispatch is a different problem.
Priority creates starvation: if scavenge always outranks investigate, the victim's faction never responds.
AlifePlus rotates the dispatch order through a round-robin cursor per cause type.
If scavenge claimed first on the last massacre, investigate goes first on the next one.
Personality already gates by faction identity, so priority-based differentiation is redundant at the consequence layer.

> "The more points with artifacts the faction controls, the cooler stalkers it can recruit."
> Clear Sky pre-release interview [5]

---

## Mechanics

Every mechanic below follows the same pattern: observe an engine event, evaluate a predicate, and dispatch a consequence using entities already living in the simulation.

GSC designed the same loop for every offline decision.
A stalker evaluates available tasks, checks equipment against encounter probability, and selects the best option.
If current equipment is insufficient, the stalker picks a different task instead of failing [8].
A death triggers witness detection, which triggers news propagation, which triggers a trader bounty [8].
An SOS signal triggers personality evaluation, which triggers rescue, murder, or silence [8].
The decision is always a function of world state, not a random roll.

> "The quantity of ammunition and medkits, as well as the type of weapon required to complete a mission, is calculated based on the probability of encountering a specific number and quality of monsters. If the Stalker's current equipment is insufficient for the mission, they will select a different mission instead."
> [8]

---

### Massacre: Scavenge, Investigate

When deaths accumulate at a location, nearby scavenger factions converge on the site and the victims' faction sends squads to investigate.

AlifePlus tracks kills per smart terrain through the massacre cause.
When the count crosses a threshold, two consequences fire: scavenge draws predator and scavenger factions toward the bodies, and investigate sends the victim's faction to find out what happened.

In shipped code, individual corpse convergence already exists.
Monsters scan for dead entities by distance (`monster_corpse_memory`), then follow a full state chain: approach, inspect, drag, eat, rest (`state_defs.h`).
Corpse locking prevents competition over the same body.
AlifePlus scales this from one body to a massacre site.

The engine also tracks kill-based attitude shifts between factions (`relation_registry`), but nothing acts on the change.
Investigate acts on it.

GSC modeled combat outcomes between all creature types through a 21x21 effectiveness matrix, where every species pairing produces a quantified result [8].
They modeled detection probability as a function of enemy health and terrain type, ranging from 1% to 100% across a 10x5 lookup table [8].
They designed a murder witness system where an NPC who sees a kill records the event, spreads it as news to every stalker encountered afterward, and transmits a report to a trader within 60 to 120 seconds [8].
If the witness is killed before the message is sent, no one ever learns of the murder [8].
Massacre sites are where all of these systems were meant to intersect: combat produces bodies, detection draws attention, and witnesses spread the news.

> "To notify player about the offline activity, we used news, which were generated if some character could see an event or its consequences."
> Dmitriy Iassenev, Game Developer (2008) [1]

---

### Squad Kill: Revenge, Flee

When the last member of a squad dies, same-faction squads pursue the killer and nearby squads of the victim's faction fall back to the nearest base.

AlifePlus fires the squadkill cause on total squad wipeout.
Two consequences dispatch: revenge sends same-faction squads after the killer through real movement, and flee routes nearby squads to the nearest friendly base through the engine's own precondition routing (`sim_board`).

X-Ray adjusts faction goodwill on kills, scaled by victim rank (`relation_registry_actions`).
It tracks who killed whom and shifts attitudes between factions.
Nothing in the base game acts on the change.
Revenge acts on it.

GSC designed a full murder consequence chain.
If a character spots the attacker before dying, the attacker's name is transmitted to a trader [8].
The trader, depending on personality, reputation, and relationship with the victim, may issue a bounty on the killer [8].
When a stalker spots the killer afterward, they report the location to traders, who broadcast it to all nearby stalkers, and the hunt begins [8].

> "The Trader issues bounties on Killer Stalkers. When a Stalker spots a Killer, they report the Killer's location to the Traders, who then broadcast this information to all other Stalkers. Consequently, all Stalkers in the immediate vicinity begin hunting down the Killer."
> [8]

---

### Base Kill: Support, Flee

When a faction base sustains enough deaths, nearby friendly squads converge to reinforce and squads at the base evacuate.

AlifePlus fires the basekill cause when kill count at a base smart terrain crosses a threshold.
Two consequences dispatch: support sends nearby same-faction squads to reinforce, and flee evacuates squads currently at the base to the nearest friendly alternative.
Both use the engine's own precondition routing (`general_base_precondition` in `sim_board`), which gates base access by faction and time of day.

Clear Sky's faction war routed reinforcements to contested control points through the same precondition system.
AlifePlus uses the same routing when a base death threshold is breached.

GSC designed bases as persistent strategic assets.
The faction leader operates as a permanent merchant from a guarded hideout, with well-equipped guards whose number and gear depend on the faction [8].
Stalkers who trespass into faction territory receive a PDA warning before being engaged without further notice [8].
If the leader is killed, the most skilled remaining member assumes leadership after the next emission [8].
Bases were worth defending because they anchored the faction's economy and command structure.

> "The in-game Zone has been shaken by furious war of factions for the new territories, resources, and control points of the important paths to the center of the Zone."
> Clear Sky official gameplay page [4]

---

### Alpha: Promote

A mutant accumulates kills and levels up through 10 tiers, gaining hit power buffs, panic immunity, and valuable loot in its inventory.

AlifePlus fires the alpha cause when a mutant killer's projected kill count crosses a new level threshold.
Stalkers are excluded at the cause level. The engine's native rank system already handles stalker progression (kills award `rank_kill_points`, rank interpolates dispersion).
The promote consequence applies hit power multipliers (outgoing bonus and incoming reduction), sets panic threshold to zero so the alpha never flees, and injects loot items from tiered pools so killing an alpha is worth the risk.

Monster rank in X-Ray is fixed at spawn from the `clsid` multiplier (`sim_offline_combat.script`).
A dog that kills fifty stalkers has the same combat power as one that never fought.
The engine's creature effectiveness tables prove GSC intended a hierarchy: their `creature_courage.txt` assigns per-species Attack scores (rats 60, bloodsucker 40, chimera 20) and Defend scores, feeding a 21x21 `CreatureEffectiveness` matrix that models outcomes across all creature type pairs [8].
Their `Monster_expedience.txt` models predation as a function of starvation and prey edibility: a starving predator encountering edible prey hunts with 100% probability [8].
GSC designed creatures as active predators with individual combat ratings and hunger-driven motivation, but the combat hierarchy was static by species, never by individual experience.
AlifePlus adds the individual axis: a mutant that survives and kills becomes an alpha, the dominant individual of its species.

> "GSC's `creature_courage.txt` assigns per-species Attack and Defend ratings across 21 creature types. The 21x21 `CreatureEffectiveness` matrix models combat outcomes for every creature pair. `Monster_expedience.txt` maps Starvation x Edibility to hunt probability (3x3 = 100% for hungry predator + edible prey). The hierarchy was designed but static by species."
> creature_courage.txt, CreatureEffectiveness, Monster_expedience.txt [8]

---

### Alpha Kill: Targeted

When an alpha dies, other alphas on the same tier pursue the killer through real movement.

AlifePlus fires the alphakill cause when an alpha mutant is killed.
The targeted consequence sends same-tier alphas after the killer.

GSC designed a trader bounty system where kills had escalating repercussions.
A trader evaluates the victim's reputation and relationship before deciding whether to issue a contract on the killer [8].
After three to five unwitnessed murders, the killer is placed on the trader's blacklist [8].
AlifePlus applies the same escalation principle to mutants: killing an alpha is not free, and the pack responds.

> "The more unwitnessed murders a player commits, the more their standing with the Trader deteriorates. After 3-5 such murders, the player is placed on the blacklist."
> [8]

---

### Wounded: Hunt, Help

When an NPC or the player uses a healing item, nearby predator mutants close in and nearby same-faction squads rush to help.

AlifePlus fires the wounded cause through a synthetic callback.
The engine has no callback for NPC medkit use, so AlifePlus hooks the healing handler and broadcasts the event.
Two consequences dispatch: hunt sends predator mutants toward the wounded target, and help sends same-faction squads to assist.

X-Ray models predator behavior in detail.
Ambush states, pursuit states, and a full corpse lifecycle exist in `state_defs.h`: approach, inspect, drag, eat, rest.
Chimeras have a dedicated hunting state machine (`chimera_state_hunting`).
Hunger was planned and abandoned: `GetSatiety()` returns hardcoded 0.5 and `ChangeSatiety()` does nothing (`base_monster.h`).
The predator states survived but the hunger drive that was meant to trigger them did not.
AlifePlus extends the target from dead prey to weakened prey.

GSC's design documents describe personality-driven rescue as a first-class mechanic.
An SOS signal triggers evaluation based on distance, potential gain, and personality [8].
The response ranges from rescue with food, medical aid, and ammunition to killing and looting the sender [8].
AlifePlus splits this into two consequences: predators respond to vulnerability and allies respond to need.

> "A complex AI system with a life simulation system that enables any monster and NPC to act independently on any game level, with creatures having a full life cycle where they hunt, feed, take rest, and sleep."
> Oles Shishkovtsov, ModDB interview [3]

---

### Harvest: Hunt

When an NPC or the player picks up an artefact, nearby outlaws pursue the carrier through real movement.

AlifePlus fires the harvest cause on the engine's item take callback (`eOnItemTake` in `InventoryOwner.cpp`), which fires for both the actor and NPCs.
The hunt consequence sends outlaw-faction squads after the carrier.

NPC artefact gathering already exists in shipped code (`xr_gather_items.script`) with community filters and timers.
Artefact spawning was planned at the engine level: `spawn_artefacts` exists in `xrServer_Objects_ALife_Monsters_script.cpp` but is commented out.
The economy was designed but the competitive response was never added.

GSC's design documents describe a full artefact economy where dealers generate quests based on anomalous activity maps, and stalkers compete for the same finds [8].
Their anomaly encounter pipeline calculates detection probability by anomaly type and detector quality, interaction probability by distance to the nearest graph point, and retreat probability by intelligence [8].
Artefacts are found in anomaly fields that the engine already models with three probability stages.
AlifePlus adds the missing response: someone else wanted that artefact too.

> "In early versions of the simulator, a character knew one or several dealers, which had sets of quests that they generated based on the map of anomalous activity and requests from organizations, which wanted to get rich from artifacts found in the Zone."
> Dmitriy Iassenev, Game Developer (2008) [1]

---

### Stash: Loot, Ambush, Fill

A squad discovers a nearby stash on smart terrain transition, walks there to loot it, camp to ambush others, or hide supplies inside.

AlifePlus fires the stash cause during radiant evaluation when a squad detects a stash within range.
Three consequences dispatch from a shared config table: loot walks the squad to the stash and takes the contents, ambush walks the squad there and waits, and fill walks the squad there and deposits items.

X-Ray indexes all inventory boxes on load and fills half with random loot (`treasure_manager.script`).
A callback for NPC interaction with inventory boxes exists but the handler is empty (`npc_on_item_take_from_box` in `axr_main.script`).
NPC stash involvement in the base game is purely narrative: surrendering NPCs reveal stash locations (`dialogs.script`), task rewards mark stashes on the map (`xr_effects.script`), and the PDA hints at contents (`ui_pda_npc_tab.script`).
The infrastructure was built and left disconnected.

GSC designed a full robbery and confiscation scheme for territorial encounters.
The sr_robbery system uses three nested space restrictors, a PDA warning sequence, a weapon-down compliance check, a dialog exchange, and per-item confiscation probabilities for money, weapons, and ammunition [8].
GSC's smart terrain classification marks some locations as resource type, meaning supply points with limited squad rotation [8].
Stashes at resource points are exactly what factions would compete over.
AlifePlus connects the infrastructure the engine already has.

> "The sr_robbery scheme operates through three nested space restrictors. A PDA command demands compliance. If the target complies, a dialog exchange processes confiscation with per-item probabilities for money, weapons, and ammunition. If not, the squad attacks without further warning."
> sr_robbery.htm [8]

---

### Area: Conquer, Decay

When a squad discovers an unguarded smart terrain nearby, it walks there and claims it on arrival.
Both stalker and mutant factions conquer territory through the same mechanism.

AlifePlus fires the area cause during radiant evaluation when a squad detects an empty smart terrain that is not a base.
The conquer consequence walks the squad there and injects the conqueror's spawn entry into the smart terrain's respawn parameters on arrival.
The injection is additive: the original spawn entries keep firing alongside the conqueror's entry.
A mutant lair conquered by duty still spawns its mutants, but duty squads also appear.

The engine's smart terrain is a bare scheduling shell with zero faction fields at the C++ level (`CSE_ALifeSmartZone` in `xrServer_Objects_ALife.h`).
All territory control lives in Lua.
The base game derives faction from physically present stalkers (`check_smart_faction`), but squads never proactively decide to claim territory.
The simulation board gates access through time windows: enemy base takeover is restricted to 03:00-06:00, territory takeover to 09:00-15:00 (`general_base_precondition`, `general_territory_precondition` in `sim_board.script`).
AlifePlus adds the proactive decision.

Mutant factions conquer through the same cause and consequence.

Conquered territory decays after 72 game hours by default.
A FIFO eviction cap prevents any faction from holding more than 50 conquered smarts.
Same-faction reconquest refreshes the timestamp.
Different-faction conquest overwrites without counting against the cap.
Decay removes the injected entry and restores the smart terrain to its original spawn tables.
The cycle creates gradual territorial oscillation: factions expand, their presence fades, other factions move in.

GSC designed faction expansion as a staged process.
Their faction settings define expansion levels with increasing squad requirements and per-faction simulation priority tables that shift targeting preferences as the faction grows [8].
Smart terrains are classified by strategic type: territory for defensible ground, resource for supply points [8].
Their creature reproduction tables assign per-species birth probability and birth speed across 21 creature types, with some species at 70% birth probability and others at zero [8].
Territory that mutants hold becomes territory where mutants breed.

> "A control point leading to a place with artifacts is controlled by bandits. Their destruction means that the path to resources is safe."
> Clear Sky pre-release interview [5]

---

### Infestation: Nest

When a mutant squad's territorial instinct fires, it walks to a species-appropriate smart terrain and claims it as a nest.
The original spawns are suppressed and replaced with the infesting species.

Feral mutants infest lairs. Predators infest lairs or buildings. Aberrant species infest buildings and underground shelters.
A per-level cap prevents any species from dominating an entire map.

The engine already has a per-creature home point system.
`CMonsterHome` gives every monster a home position with min, mid, and max radii.
During rest, the creature returns to its home point (`eStateRest_MoveToHomePoint`).
During panic, it retreats toward home (`eStatePanic_HomePoint_Camp`).
During combat, it returns if the fight drifts too far (`eStateAttack_MoveToHomePoint`).
This operates at the individual NPC level within a single smart terrain.
AlifePlus extends it to the simulation level: a squad claims a smart terrain, and the spawn tables shift to reflect it.

GSC designed territory patrol for specific species.
Boar "патрулирует свою территорию" (patrols its territory) (monstry.doc).
Chernobyl dogs are "крайне агрессивна, патрулирует свою территорию" (extremely aggressive, patrols its territory) (monstry.doc).
Their creature reproduction tables (`BirthProbability`, `BirthSpeed`, `BirthPercentage`) assign non-zero values to only three of 21 creature types: rats (70%), rat wolves (60%), and dogs (50%) [8].
These are the pack species that patrol territory. All others have BirthProbability=0.
Territory that mutants hold is territory where mutants breed.

> "Крайне агрессивна, патрулирует свою территорию, нападает на любых противников."
> (Extremely aggressive, patrols its territory, attacks any opponents.)
> monstry.doc, Chernobyl dog entry

---

### Needs: Drives, Consequences

When a squad transitions at a smart terrain, it evaluates nine drives and pursues whichever is strongest.

AlifePlus scores each drive using Hull's Drive Reduction Theory (1943) weighted by Maslow's hierarchy of needs (1943).
The formula is `drive = weight * (elapsed / threshold)^2`.
Weights follow Maslow: survival outweighs security outweighs belonging.
The squared deprivation ratio adapts Hull: drive grows nonlinearly with deprivation, so mild discomfort is noise and critical need dominates.
Hull and Maslow were the dominant behavioral models when GSC designed A-Life in 2002-2004.

The nine drives and what happens when they win:

- **Hunger** finds a campfire and eats from inventory: bread, sausage, canned goods.
- **Sleep** waits for nightfall, finds a campfire, and sleeps through the night.
- **Rest** finds a campfire, smokes a cigarette, has a drink.
- **Heal** returns to a friendly base and uses a medkit, bandage, or stimpack.
- **Shelter** returns to a friendly base when exposed too long.
- **Money** searches anomaly fields for artefacts or hunts mutant lairs.
- **Supply** visits a trader and exchanges an artefact for ammo, grenades, or medical supplies.
- **Job** guards the base, explores the Zone, researches anomalies. Monolith and Greh worship. Military and Duty drill.
- **Social** finds a campfire or returns to base, shares cigarettes and drinks.

NPCs consume inventory items on arrival: food, cigarettes, drinks, medkits, or artefacts depending on the need.

X-Ray has all the building blocks but no algorithm connecting them.
`GetSatiety()` returns hardcoded 0.5 and `ChangeSatiety()` does nothing (`base_monster.h`): hunger was planned and abandoned.
`eStateEat` defines the full eating state chain and `eStateRest` defines sleep (`state_defs.h`): the states exist but nothing triggers them by need.
Campfire social dynamics exist at smart terrains (`sr_camp.script`).
Job assignment for campfire, rest, and guard roles exists (`gulag_general.script`).
NPC healing exists (`xr_eat_medkit.script`).
AlifePlus provides the algorithm that connects needs to behavior.

GSC's design documents describe the full offline lifecycle where personal status management interrupts task execution.
If health is low, the stalker uses a medkit.
If hungry, the stalker eats food or drinks vodka.
If fatigued, the stalker lies down to rest [8].
Their job system assigns roles per time of day: guards and patrols during the day, sleep in designated beds at night, rangers during weather transitions, and defenders during anomalous conditions [8].
Their surge death probability table ranges from 1% on sheltered terrain to 85% on open ground [8], which is why shelter exists as a drive.

`GetSatiety()` returning 0.5 proves hunger was planned.
`eStateEat` proves eating was planned.
Walking to a campfire hungry and eating nothing is fake.
The artefact economy described in [1] required exchange.
Free items from a trader is a spawn, not a trade.

> "I would very much like to play a game in which characters would live their own lives, each would have his own goal in the game, each would have human-like (or some specific for monsters) needs that the character would have to satisfy."
> Dmitriy Iassenev, Game Developer (2008) [1]

> "It was planned that character behaviour would diversify due to introduction of requirements of food and sleep."
> Dmitriy Iassenev, Game Developer (2008) [1]

---

### Instincts: Feed, Sleep, Explore, Socialize, Infest

Mutants develop instincts that drive movement the same way stalker needs do: Hull scoring picks the strongest unmet drive, and the consequence routes the squad to an appropriate smart terrain.

AlifePlus fires the instincts cause during radiant evaluation when a mutant squad's drives are overdue.
Four instincts dispatch, each gated by species behavioral alignment:

**Feed** routes all species to open territory.
GSC designed every creature with the same feeding state machine (`eStateEat` in `state_defs.h`): approach, inspect, drag, eat, rest.
Every species registers the same `can_eat()` function (`monster_state_manager_inline.h:53`).
AlifePlus puts predators in proximity with prey through shared territory.
The engine handles combat and corpse consumption when they meet.

**Sleep** returns the squad to a species-appropriate rest location.
Cowardly species sleep in open fields.
Feral species sleep in lairs.
Predators sleep in lairs or buildings.
Aberrant species sleep in buildings and underground shelters.
GSC documented these habitats per species: bloodsucker "mesto zhitel'stva razvaliny, podzemel'ya, broshennye postroyki" (residence: ruins, dungeons, abandoned buildings) (monstry.doc).
Burer "zhivet tol'ko v mrachnykh, temnykh podzemyel'yakh" (lives only in dark dungeons) (monstry.doc).

**Explore** moves the squad to a different territory or lair.
Cowardly, feral, and predator species explore.
Aberrant species do not explore -- they are lair-bound.
GSC documented this sedentary nature explicitly: poltergeist "na otkrytykh mestakh ne vstrechayetsya" (never found in open areas) (monstry.doc).
Burer never leaves its dungeon.
Controller "predpochitayet zasady v zdaniyakh" (prefers ambushes in buildings) (monstry.doc).
Species that never leave their lair have no reason to wander.

**Socialize** draws the squad toward smart terrains where same-faction squads are present.
Cowardly and feral species socialize -- they are pack and herd animals.
Boar is "staynoye zhivotnoye" (herd animal) with leader morale mechanics (monstry.doc).
Dogs are "agressivny, protivnika okruzhayut" (aggressive, surround the enemy) -- pack tactics imply social structure (monstry.doc).
Flesh form "staya" (pack) and flee together (monstry.doc).
Predator and aberrant species do not socialize -- they are solitary hunters and lair defenders.
Bloodsucker ambushes from cover alone.
Chimera "pryachetsya, obkhodit so spiny" (hides, flanks from behind) -- solitary ambush predator (monstry.doc).

The engine already models the complete feeding state machine.
`eStateEat` defines approach, inspect, drag, eat, and rest substates (`state_defs.h`).
`CorpseMan` tracks nearby corpses in memory.
`hungry()` returns true after 20 seconds without food.
AlifePlus does not simulate eating -- it puts predators in proximity with prey through shared territory.
The engine handles combat and corpse consumption when they meet.

GSC designed creatures with species-specific feeding behaviors.
Dogs hunt rats and flesh (`monstry.doc: "okhota na krys i plot', otdykh"`).
Controllers lure small animals for food (`monstry.doc: "zhivotnykh podmanivayut i ubivayut dlya propitaniya"`).
Bloodsuckers eat everything that moves (`monstry.doc: "est vse chto dvizhetsya"`).
The feeding drive was planned and stubbed (`GetSatiety()` returns hardcoded 0.5, `ChangeSatiety()` does nothing in `base_monster.h`) but the motivation layer that triggers feeding behavior was never connected.
AlifePlus provides that motivation.

> "A complex AI system with a life simulation system that enables any monster and NPC to act independently on any game level, with creatures having a full life cycle where they hunt, feed, take rest, and sleep."
> Oles Shishkovtsov, ModDB interview [3]

---

### Day/Night Cycle

Stalkers and mutants follow a day/night activity cycle.
During their active period, creatures feed, explore, work, and socialize.
During their dormant period, they seek shelter and sleep.

Stalkers are diurnal: active during the day (05:00-20:00), dormant at night.
Nocturnal mutant species -- bloodsuckers, lurkers, chimeras, zombies, fractures -- are active at night (20:00-05:00) and sleep during the day.
All other species are diurnal.

The engine already encodes this distinction in its faction system.
`monster_predatory_day` and `monster_predatory_night` are separate factions with different spawning and simulation behavior.
`monster_zombied_day` and `monster_zombied_night` follow the same split.

GSC's gulag job system defines four states per smart terrain: Day (6:00-21:00), Night (21:00-6:00), Attack, and Defense.
Day jobs include camp, walker, patrol, guard, and sniper.
Night jobs replace camp with sleep.
Attack and defense override the day/night rule [8].

> "Raboty universal'nykh lagerey. U kazhdogo universal'nogo lagerya yest' 4 sostoyaniya: 1. Den' (6:00-21:00). 2. Noch' (21:00-6:00). 3. Ataka. 4. Oborona."
> (Universal camp jobs. Each universal camp has 4 states: 1. Day. 2. Night. 3. Attack. 4. Defense.)
> Smart_terrain_jobs.htm [8]

GSC described nocturnal hunting explicitly for bloodsuckers.
During the day they ambush from dark places and shelter.
At night they emerge into the open to hunt.

> "Noch'yu i v temnote luchshe vidit. Noch'yu vykhodit na otkrytyye mesta, okhotit'sya."
> (Sees better at night and in darkness. At night goes to open areas to hunt.)
> monstry.doc, bloodsucker entry

> "Mesto zhitel'stva razvaliny, podzemel'ya, broshennye postroyki."
> (Place of residence: ruins, dungeons, abandoned buildings.)
> monstry.doc, bloodsucker entry

AlifePlus implements the same split through the activity alignment axis.
The `is_active_period` function resolves the current period for any species or community.
Every drive entry declares whether it requires the active or dormant period.
The same mechanism applies uniformly to stalker needs and mutant instincts.

---

## Alignment

GSC classified characters along a moral axis with nine types across two dimensions: principled, self-serving, or unprincipled crossed with good, cold-blooded, or evil (Ai.doc:65-76, тип характера).
Each type defines specific behavioral rules.
None of it shipped.

> "Principled Good: Saves everyone, with the exception of enemies. Tries to avoid killing mutant animals whenever possible. Deliberately eliminates humanoid mutants. Would never commit an act of betrayal. Does not loot abandoned equipment. Does not seek revenge, though does not forgive enemies."
> [8]

> "Principled Evil: Will kill and rob those in distress; may even deliberately head toward a distress signal for this very purpose. The pursuit of revenge takes precedence over all other objectives. Sneaks up on neutral parties and attacks them. Does not betray friends."
> [8]

> "Neutral Evil: If profitable, will kill and loot those in distress. If profitable, uses PDA signals to lure others into a trap. If the gain is substantial, may betray friends. If profitable, helps neutral parties when they are in distress, then attacks them."
> [8]

Three character types, three completely different responses to the same event.
GSC treated mutants as actors in the same system and classified creatures into four groups by nature (monstry.doc):

1. Ordinary animals (rats).
2. Animal mutants (boar, dogs, pseudodog, flesh, chimera).
3. Humanoid mutants (zombie, controller, burer, fracture).
4. Aberration mutants (bloodsucker, pseudogiant, poltergeist).

With a documented fear hierarchy between groups:

> "Vse troe lyudi-mutanty v kotorykh mnogoe ostalos' ot cheloveka. Vse troe izbegayut sleduyushchuyu gruppu polnykh urodov-mutantov: krovososa, psevdogiganta, poltergeista."
> (All three humanoid mutants retain much of their humanity. All three avoid the next group of complete freak-mutants: bloodsucker, pseudogiant, poltergeist.)
> monstry.doc, group 3 -> group 4 relation

The engine collapsed all of this into six factions by time-of-day activity.
A dog, a pseudodog, a psy dog, and a lurker share `monster_predatory_day`.
A snork, a zombie, a fracture, and a controller share `monster_zombied_day`.
The faction called `monster_vegetarian` contains boar and flesh, both of which GSC's design documents describe as corpse-eaters.

> "Pozhirayut trupy ubitykh."
> (Devour corpses of the killed.)
> monstry.doc, boar entry

> "Est trupy krys, sobak, lyudei."
> (Eats corpses of rats, dogs, people.)
> monstry.doc, flesh entry

The engine confirms this.
Every creature shares the same `can_eat()` function (`monster_state_manager_inline.h:53`).
Every state manager -- boar, flesh, dog, snork, tushkano, cat, chimera, controller, burer, fracture, bloodsucker, gigant, zombie -- registers `eStateEat` with identical corpse-detection logic.
There is no diet parameter, no food-type filter, no herbivore flag.
The only creature with eat disabled is poltergeist (`poltergeist_state_manager.cpp:75`, commented out).
There are no herbivores in the Zone.

GSC designed distinct feeding behaviors per species.
Dogs hunt rats and flesh (`monstry.doc: "okhota na krys i plot', otdykh"`).
Controllers lure small animals for food (`monstry.doc: "zhivotnykh podmanivayut i ubivayut dlya propitaniya"`).
Bloodsuckers eat everything that moves (`monstry.doc: "est vse chto dvizhetsya"`).
The engine gave them all the same stomach.

AlifePlus maps GSC's moral axis to factions as a hard filter for stalkers.
Principled factions (Duty, Military, Monolith, ISG) follow orders and hold positions.
Self-serving factions (Loners, Clear Sky, Ecologists, Freedom, Mercs) act on their own goals.
Outlaw factions (Bandits, Renegades, Sin) operate outside any code.
Mutant alignment operates at species level, not faction level, restoring the per-creature behavioral identity GSC documented but the engine erased.
Four alignment categories map the behavioral axis for mutants the same way principled/selfserving/outlaw maps the moral axis for stalkers.

Cowardly species flee danger and scavenge what others leave behind.
Flesh is "podlaya, khitraya i truslivaya, mnogikh boitsya" (cowardly, cunning, timid, fears many) (monstry.doc).
Zombie exists in "rastitel'noye sushchestvovaniye" (vegetative existence) (monstry.doc).
Tushkano, rat, and karlik round out the bottom of the food chain.

Feral species run in packs, defend their own, and avenge fallen members.
Boar is "staynoye zhivotnoye" (herd animal) with leader morale mechanics (monstry.doc).
Dogs are "agressivny, protivnika okruzhayut" (aggressive, surround the enemy) (monstry.doc).
Gigant has "povadki buyvola" (buffalo-like behavior) -- brute force, not psychic (monstry.doc).
Pseudodog, snork, and cat complete the group.

Predators hunt alone, ambush from cover, and pursue wounded prey.
Bloodsucker "est vse chto dvizhetsya" (eats everything that moves) and "napadayet iz temnykh mest i ukrytiy" (attacks from dark places and cover) (monstry.doc).
Chimera "pryachetsya, obkhodit so spiny" (hides, flanks from behind) (monstry.doc).
Lurker, psysucker, and fracture share the same hunting pattern.

Aberrant species operate through psychic abilities and lair defense.
Controller uses "telepaticheskoye vozdeystviye" (telepathic influence) and "predpochitayet zasady v zdaniyakh" (prefers ambushes in buildings) (monstry.doc).
Burer "zhivet tol'ko v mrachnykh, temnykh podzemyel'yakh" (lives only in dark dungeons) (monstry.doc).
Poltergeist "na otkrytykh mestakh ne vstrechayetsya" (never found in open areas) (monstry.doc).
Psy dog generates psychic phantoms to hunt by proxy.

These four categories form a linear food chain. Cowardly species scatter from feral, predator, and aberrant. Feral species scatter from predator and aberrant. Predators scatter from aberrant. Aberrant species fear nothing.
GSC documented this hierarchy implicitly: flesh "mnogikh boitsya" (fears many), chimera "biologicheskikh vragov malo" (few biological enemies), controller and burer avoid bloodsuckers and pseudogiants (monstry.doc group 3->4 relations).
AlifePlus makes it explicit as a scatter instinct: when a lower-tier mutant sees a higher-tier predator within eye range, it relocates to a safe location.

The engine's `player_id` gates the question "is this a mutant at all" for zero luabind cost.
Species identity resolves through `xcreature.get_mutant_species` only when the consequence needs to know what kind of mutant it is.

Alignment determines what a faction or species can do at all.
Military never flees a base attack.
Ecologists never conquer territory.
Renegades never investigate massacres.
Outlaws never help the wounded.
Cowardly mutants never avenge or hunt.
Feral mutants never ambush stashes.
These are structural constraints, not tunable probabilities.

---

## Personality

GSC modeled personality with five independent attribute axes that fed probability lookup tables [8]:

- **PersonalAggressiveness** (1-5): willingness to engage.
- **PersonalGreed** (1-5): profit motivation.
- **PersonalIntelligence** (1-5): retreat decision quality.
- **PersonalEyeRange** (1-5): awareness radius.
- **PersonalRelation** (1-2): friend or enemy.

The Expediency function combined these into a four-dimensional lookup: Relation x Cost x Greed x Aggressiveness -> action probability between 5% and 65% [8].
The decision is a function of who you are, not a coin flip.

> "Each faction possesses a distinct 'personality' (either pre-assigned or randomly generated); based on this personality, the faction determines which other factions it will initially treat as allies or enemies."
> [8]

> "A character's personality determines what they prioritize as more valuable: money or reputation points (including negative reputation points)."
> [8]

The EFC (Evaluated Function Container) lookup tables in the original design archives prove this was not abstract. GSC built concrete probability matrices: `EnemyDetectProbability` is a 10x5 table indexed by `EnemyDetectability` and `PersonalEyeRange`. `EnemyRetreatProbability` crosses `EnemyDetectability` with `PersonalIntelligence`. `Expediency` crosses four axes into a single action probability. These are not descriptions of intent -- they are filled tables with numeric values that a running simulation would evaluate per tick.

AlifePlus implements personality as a probability layer that runs after alignment.
Seven traits per stalker faction, five per mutant species, each tracing directly to GSC's EFC variable names: `PersonalAggressiveness` -> aggression, `PersonalGreed` -> greed, `PersonalIntelligence` -> discipline, `PersonalEyeRange` -> perception, `PersonalRelation` -> relation.
Two additional traits -- territory (from `CMonsterHome` territory system) and survival (biological needs drive) -- complete the set.
Stalker factions and mutant species share the same gate function and the same roll mechanic.
The decision is probabilistic given the identity and the event, but only within the set of actions alignment permits.

Each trait value is a direct probability (0.0-1.0) grounded in GSC lore. Trait values are pure: no weight multiplier, no per-consequence scaling. Each consequence declares at most two traits. The check averages those traits and rolls against the result, clamped to a user-configurable floor and ceiling (MCM `personality_min` and `personality_max`, defaults 0.20 and 0.70). The clamp ensures that even unfavorable factions act occasionally (floor) and even favorable factions fail sometimes (ceiling).

Survival is a flat band (0.40-0.60) across all factions and species. Eating and sleeping are universal biological drives, not faction differentiators. The interesting personality differences come from the other traits.

Inverted traits (`INV_`) express behaviors driven by the absence of a quality. Fleeing a base attack is gated by `INV_DISCIPLINE` and `INV_TERRITORY` -- low discipline and low territorial attachment make flight more likely. The check resolves `1 - base_value` before averaging.

For stalkers, personality keys on faction community.
For mutants, personality keys on species resolved through `xcreature.get_mutant_species`.
A dog and a lurker both belong to `monster_predatory_day`, but a dog has high relation (pack loyalty) and moderate aggression, while a lurker has low relation (solitary) and high aggression.
The engine faction cannot express this.
GSC's design documents gave each creature its own behavioral profile, and AlifePlus restores that granularity.

Aggression gates revenge, alpha hunt, wounded hunt, territory conquest, harvest robbery, and stash ambush.
Greed gates stash loot, stash fill, money harvest, supply runs, and harvest robbery.
Perception gates massacre investigation, exploration, research, money harvest, and mutant feeding.
Territory gates area conquest, shelter, outpost duty, and exploration.
Relation gates squad revenge, massacre investigation, base support, wounded help, social behavior, and supply trading.
Discipline gates base support, shelter, outpost duty, research, and healing -- and its inverse gates fleeing.

---

## Range: Awareness and Reach

GSC's `PersonalEyeRange` was a per-entity attribute that determined how far a stalker could perceive threats. It fed directly into `EnemyDetectProbability`: a stalker with high EyeRange detected enemies that others missed [8]. The range was not a game mechanic exposed to the player -- it was an internal simulation parameter that made each entity's awareness feel distinct.

AlifePlus extends this into two range tiers that govern how far consequences search for targets:

- **EyeRange** (200m): line-of-sight. What the squad can see from where it stands. A stalker arriving at a smart terrain spots a nearby stash, an unclaimed outpost. A mutant pack sees prey at a watering hole.
- **RadioRange** (500m): PDA radio. A stalker hears about a massacre, a base under attack, a trader location. Information travels further than sight.
- **ScentRange** (500m): scent tracking. A bloodsucker smells corpses from 500m. A pack follows pheromone trails. A predator tracks wounded prey on the wind. Same distance as radio, different sense, independently tunable.

The 200m EyeRange is not arbitrary. Empirical measurement of smart terrain spacing across 14 Anomaly levels shows that the median nearest-neighbor distance between smarts ranges from 44m (dense urban) to 148m (sparse industrial), with 200m covering the 90th percentile on 93% of levels.

Each system maps to a range based on how the squad becomes aware:

- **Opportunities** use EyeRange. A squad arrives and sees what's nearby. Stash, unclaimed territory. If nothing is visible, nothing happens. This is local and opportunistic.
- **Reactions** use RadioRange (stalkers) or ScentRange (mutants). A stalker hears about a kill over radio. A mutant smells blood. The event happened elsewhere; the squad responds from a distance.
- **Needs** use RadioRange. A hungry stalker knows where campfires are from PDA contacts. A wounded stalker knows the nearest medic.
- **Instincts** use ScentRange. A hungry predator tracks scent to prey. A pack follows pheromone trails to a gathering point.

Three tiers create a natural boundary between seeing, hearing, and smelling. You act on what you see, respond to what you hear, hunt what you smell. This is how GSC designed awareness: `PersonalEyeRange` determined the visible world, while faction SOS signals carried information beyond it.

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
- [6] OpenXRay (x-ray-16)
  https://github.com/OpenXRay/xray-16
- [7] xray-monolith (Anomaly engine, modded exes)
  https://github.com/themrdemonized/xray-monolith
- [8] GSC Game World internal AI design documents (pre-release design phase, circa 2002-2006). Three archives: Ai.rar (main AI design doc, monster FSMs, combat spreadsheets), Help.rar (A-Life help system, faction settings, Data.save lookup tables), Tools.rar (offline simulation spreadsheets, trader design). Originally preserved on the OLR (Oblivion Lost Remake) forums and publicly circulated within the STALKER modding community.
  - **Reached engine:** morale system (`CMonsterMorale`), monster home/territory (`CMonsterHome`), monster FSM states (`state_defs.h`), squad coordination (`ai_monster_squad`), `monster_community` relation tables, `EnemyDetectability` in creature configs. Partially: `eStatePanic` exists but suppressed by near-zero `panic_threshold` configs, `eStateEat` exists but `GetSatiety()` returns hardcoded 0.5.
  - **Reached engine as dead code:** NPC-NPC encounter hook (`alife_monster_abstract.cpp`, marked `#pragma todo("Do not forget to uncomment here!!!")`), `spawn_artefacts` in alife objects (commented out), anti-aim ability (implemented but counterproductive).
  - **Design docs only, never coded:** personality axes (PersonalAggressiveness/Greed/Intelligence/EyeRange/Relation), Expediency 4D function, 8-type moral classification, SOS response logic, offline stalker FSM (task selection, equipment calculation, personal status management), murder/witness/bounty chain (suspect system, 50m radius, trader blacklist), faction personality concept, sr_robbery confiscation scheme, trader reputation/blacklist system, creature reproduction tables (BirthProbability, BirthSpeed per species), 21x21 creature effectiveness matrix, creature courage ratings (per-species Attack/Defend scores), monster expedience function (Starvation x Edibility -> hunt probability), anomaly encounter pipeline (3-stage probability calculation), surge death probability by terrain type, EntityCost valuation matrix, faction expansion levels with sim_prior tables.
