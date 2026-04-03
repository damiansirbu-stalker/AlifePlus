AlifePlus: Emergent A-Life for STALKER Anomaly, by Damian
GitHub: https://github.com/damiansirbu-stalker/AlifePlus
Changelog: doc/changelog

! Please use the RESET button in MCM when updating to a new version !

You are not special.

AlifePlus is a behavior mod for STALKER Anomaly that turns A-Life into a real simulation again: NPCs and mutants act independently, pursue goals, react to threats, and create knock-on effects that persist whether the player is present or not. It is inspired by Roadside Picnic and the original STALKER vision: the Zone runs on its own rules, and the actor is just another entity inside it.

Every action has a systemic cause. Nothing moves without a reason. The simulation runs whether you are there or not.

The mod is built around two invariants:
- Event-driven contract: nothing runs unless the engine says something happened.
- Physical simulation guarantee: nothing is spawned, teleported, or fabricated to fill a role. Consequences use entities that already exist in the simulation.

The economy uses real inventory items, and real needs drive real decisions.

The Legacy Approach:
Many A-Life mods increase activity by forcing movement on a schedule: scan the world, pick squads, push them toward scripted goals, repeat. O(n) per tick, ticking every frame.
They hijack squads and overwrite important variables used by the engine or other mods. Heavy loops with API calls cause ghosting, leaking, or save corruption.
Behaviors like trading have no cohesive economy. The world feels more alive because things move around, but the movement has no systemic purpose and the performance cost adds up.

The AlifePlus Approach:
Engine events are intercepted as they fire. The system reacts when something happens, not on a timer. If nothing is happening in the simulation, AlifePlus is idle.
Squads are extended, not hijacked. The engine's own job system produces the correct behavior without overwriting its variables.

All engine interaction goes through xlibs, a shared library reverse-engineered from X-Ray Monolith C++ source covering squad and smart terrain APIs, event bus, synthetic callbacks, crash-safe object resolution, and microsecond profiling.
Where the engine lacks a callback, a runtime hook emits a synthetic event without modifying base scripts.
Code patterns validated against Demonized, Vintar0, RavenAscendant, xcvb.
No base script edits, no engine patches. Runtime callbacks and hooks only.

Rate-limited event pipeline with token budgets and cooldowns, with separate pacing for radiant and reactive streams. Designed to prevent the ghosting and save corruption caused by squad hijacking and engine API spam.
This keeps long saves stable by avoiding permanent squad control and limiting engine pressure under load.

Events chain deterministically. Causes dispatch consequences, and consequences change the world state, which triggers new causes. Behavior is the result of independent actors colliding in a shared simulation.
Causes are either reactive (triggered by combat, wounds, deaths) or radiant (triggered by movement, location, and needs).

Example scenario (systemic interaction):
A stalker reaches a location. AlifePlus intercepts that event.
He spots a stash ("stash" cause) and decides to loot it ("loot" consequence), not knowing if anything is inside.
He walks the whole distance to the stash but gets ambushed by another stalker who saw it first and decided to rob people ("ambush" consequence).
The player was heading to that stash and sees the fight, so he snipes both of them.
This mass killing ("massacre" cause) draws scavengers to the bodies ("scavenge" consequence) and the victims' faction sends squads to investigate ("investigate" consequence).
On the way there, the scavengers get killed by loners who were out hunting ("job" consequence).
Those loners are now tired ("needs" cause) and head back to base to smoke a cigarette and tell the story ("social" consequence).
None of this was scripted. The player walked into something already in motion.

Example scenario (escalation chain):
An artefact spawns. A stalker needs money ("needs" cause) and heads there to pick it up ("money" consequence).
On the road he fights creatures and gains enough kills to become an elite ("elite promote" consequence).
Now stronger but wounded, he bleeds ("wounded" cause) and a chimaera senses weakness and chases him down ("wounded hunt" consequence).
The chimaera kills him and gains enough kills to become an elite itself ("elite promote" consequence), a real problem in the area.
If the player kills it, he collects a bounty ("elite bounty" rewards the player). If an NPC kills it, the NPC may become an elite ("elite bounty" also rewards NPCs).
The cycle continues, the Zone does not care.

Performance & Architecture:
The mod was designed performance-first, load tested with my own testing framework (1000x load) and profiled with anomaly-devtools v1.3.0. The performance impact is near-zero as seen in the screenshots.
The entire pipeline is instrumented with OTel-style tracing, structured logging with rotation, and microsecond profiling from the engine's own timer, so every operation is measurable.

AlifePlus operates above the action planner at the simulation layer, routing squads to the correct smart terrain through SIMBOARD.
The engine's own gulag jobs, GOAP planner, and scheme bindings handle all behavior at the destination. It already knows how to make guards patrol, sleepers sleep, collectors collect. AlifePlus provides the intent, the engine provides the behavior.

[BENCHMARKS: two screenshots side by side]

Configuration:
Each cause and consequence is a module you can enable or disable through MCM. Gameplay actions (item consumption, stash looting, artefact trading, rank progression) each have their own toggle and tunable values -- chances, cooldowns, thresholds, quantities, rate limits, and budgets.
Log level goes from silent to full tracing with pathing, performance timing, and PDA map markers.

Features: causes and consequences that aggregate into hundreds of combinations

Massacre (reactive)
  Scavenge - Nearby scavengers and predators converge on the massacre site.
  Investigate - Victims' faction sends nearby squads to investigate.

SquadKill (reactive)
  Revenge - Same-faction squads pursue the killer as it moves.
  Flee - Nearby squads of the victim's faction fall back to the nearest base.

BaseKill (reactive)
  Support - Nearby friendly squads rush to reinforce the base under attack.
  Flee - Squads at the base evacuate to the nearest friendly position.

Elite (reactive)
  Promote - NPCs accumulate kills to level up. 10 levels, each requiring more kills.
  Rank boost, grenades, and AP ammo scale with level.

EliteKill (reactive)
  Bounty - Killer receives rubles scaled by the elite's level.
  Targeted - Other elites on the same tier hunt the killer across the map.

Wounded (reactive)
  Hunt - Predator mutants sense weakness and close in.
  Help - Nearby same-faction squads rush to help the wounded.

Harvest (reactive)
  Hunt - Nearby outlaws pursue whoever picked up the artefact as it moves.

Stash (radiant)
  Loot - Stalker squad spots the stash, walks there, and loots it.
  Ambush - Outlaw squad spots the stash and camps it. The stash is bait.
  Fill - Stalker squad spots the stash and hides supplies inside.

Area (radiant)
  Conquer - Squad claims the empty smart terrain. Engine spawns their faction there.

Needs (radiant)
  Stalkers have human needs. Nine Maslow-Hull drives scored by deprivation.
  The strongest unmet need wins.

  Hunger - Finds a campfire. Eats what he is carrying - bread, sausage, canned goods etc.
  Sleep - Waits for nightfall. Finds a campfire. Sleeps through the night.
  Rest - Finds a campfire. Smokes a cigarette, has a drink.
  Heal - Finds a safe location. Uses a medkit, bandage, or stimpack.
  Shelter - Finds a safe location when exposed too long.
  Money - Searches anomaly fields for artefacts, or hunts mutant lairs.
  Supply - Visits a trader. Trades an artefact for AP ammo, grenades, or medical supplies.
  Job - Guards outposts and checkpoints. Explores the Zone. Researches anomalies. Loners hunt. Monolith and Sin worship at remote sites. Military and Duty drill at forward positions.
  Social - Finds a campfire or a safe location. Shares cigarettes and drinks.

  NPCs consume real items from their inventory on arrival. A guard smokes a cigarette on duty. A hungry stalker eats food he was carrying. 
  A supply run costs some trophies, booze, an artefact and returns usables, medkits, ammunition - including AP rounds and grenades.

Requirements:
Anomaly 1.5.3
Modded exes
xlibs (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
MCM

Install (MO2):
1. Install xlibs
2. Install AlifePlus
3. Load order does not matter
4. Configure via MCM

Uninstall (MO2):
Disable or remove in MO2.

FAQ:

How do I make events occur faster or slower?
    Calmer world: raise A-Life Interval to 8-10, lower Consequence Budget to 1, raise Radiant Cooldown to 15-20.
    Busier world: lower A-Life Interval to 2-3, raise Consequence Budget to 4, lower Radiant Cooldown to 5.
    Gung-Ho: lower A-Life Interval to 1, raise Cause and Consequence Budget to max, raise consequence chances to 50+, set Radiant Cooldown to 0. Good luck!

Does it conflict with the vanilla task system?
  No. AlifePlus validates every entity through a 4-layer protection gate before interaction.
  Task givers, companions, active quest squads, and externally scripted squads are protected.

Does it work with GAMMA?
  Built and tested with GAMMA, also tested partially with Zona and EFP.
  Works directly with X-Ray engine callbacks without modifying base scripts.

Does it work with other A-Life mods?
  No known incompatibilities, including with warfare and AI addons, but will conflict behavior-wise with mods doing a lot of scripting on squads.

Does it work mid-save?
  Yes.

Do I need offline combat enabled?
  No. AlifePlus never requires or prefers online simulation.
  It works with all offline settings, including fully disabled.
  Offline combat produces fewer combat events but the system generates
  other events regardless, so A-Life activity continues.

Known Issues:
Map markers are a debugging feature. They lack persistence and do not update on entity death, despawn, or level transition.

Development:
Written against X-Ray Monolith engine source, Demonized exes source code, and Anomaly 1.5.3 unpacked gamedata.
Validated by a custom multi-stage pipeline: luacheck, selene, tree-sitter AST analysis, ast-grep structural patterns,
contract rules (API safety, cross-file dependencies, cyclomatic complexity, coding standards),
lua54 integration testing with X-Ray engine stubs, gitleaks and trufflehog secret scanning.
Full report in doc/test-report.log.

Credits:
Stalker_Boss - Russian translation

Usage and License:
  Modpacks: allowed and encouraged. Keep the readme and license files.
  Addons, patches, integrations: allowed. Credit "AlifePlus by Damian Sirbu" visibly on your mod page.
  Reproducing the implementation in other software: not allowed, even with credit.
  Full license in LICENSE file and on GitHub.
