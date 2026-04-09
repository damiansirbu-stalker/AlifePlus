AlifePlus: Emergent A-Life for STALKER Anomaly, by Damian
- GitHub: https://github.com/damiansirbu-stalker/AlifePlus
- Integration guide: https://github.com/damiansirbu-stalker/AlifePlus/blob/main/doc/integration-guide.md
- Design manifesto: https://github.com/damiansirbu-stalker/AlifePlus/blob/main/doc/manifesto.md
- Changelog: https://github.com/damiansirbu-stalker/AlifePlus/blob/main/doc/changelog

! Please use the RESET button in MCM when updating to a new version !

You are not special.

AlifePlus is a reactive alife framework for STALKER Anomaly - a complete simulation layer that replaces passive A-Life with event-driven emergent behavior. It intercepts engine events, classifies them into causes through world-state predicates, and dispatches consequences that change the simulation. NPCs and mutants act independently, pursue goals, react to threats, and create knock-on effects that persist whether the player is present or not. Squads investigate massacres, hunt artefact carriers, claim empty territory, and act on hunger, sleep, and social needs - whether you are there or not. Everything that happens to the player happens to NPCs and mutants alike. Nothing is random - every action traces back to a cause in the simulation.

Inspired by Roadside Picnic and the original STALKER vision: the Zone runs on its own rules, and the actor is just another entity inside it.

Every action has a systemic cause. Nothing moves without a reason. The simulation runs whether you are there or not.

The mod is built around two invariants:
- Event-driven contract: nothing runs unless the engine says something happened.
- Physical simulation guarantee: nothing is spawned, teleported, or fabricated. Consequences use entities that already exist in the simulation.

The economy uses real inventory items, and real needs drive real decisions.

---

Example scenario (systemic interaction):

- A stalker reaches a location. AlifePlus intercepts that event.
- He spots a stash ("stash" cause) and decides to loot it ("loot" consequence), not knowing if anything is inside.
- He walks the whole distance to the stash but gets ambushed by another stalker who saw it first and decided to rob people ("ambush" consequence).
- The player was heading to that stash and sees the fight, so he snipes both of them.
- This mass killing ("massacre" cause) draws scavengers to the bodies ("scavenge" consequence) and the victims' faction sends squads to investigate ("investigate" consequence).
- On the way there, the scavengers get killed by loners who were out hunting ("job" consequence).
- Those loners are now tired ("needs" cause) and head back to base to smoke a cigarette and tell the story ("social" consequence).
- None of this was scripted. The player walked into something already in motion.

Example scenario (escalation chain):

- An artefact spawns. A stalker needs money ("needs" cause) and heads there to pick it up ("money" consequence).
- On the road he fights creatures and gains enough kills to become an elite ("elite promote" consequence).
- Now stronger but wounded, he bleeds ("wounded" cause) and a chimaera senses weakness and chases him down ("wounded hunt" consequence).
- The chimaera kills him and gains enough kills to become an elite itself ("elite promote" consequence), a real problem in the area.
- If the player kills it, he collects a bounty ("elite bounty" rewards the player). If an NPC kills it, the NPC may become an elite ("elite bounty" also rewards NPCs).
- The cycle continues, the Zone does not care.

What you'll notice:
- Squads rerouting after nearby combat deaths
- Scavengers and investigators converging on massacre sites
- Stash traffic: NPCs walking to stashes, looting, ambushing, filling
- Outlaws hunting whoever picked up an artefact
- Retaliation and retreat loops after squad wipes
- Territory flipping as squads claim empty outposts
- Elite NPCs emerging from combat, accumulating kills and better gear
- Predators closing in on wounded targets, allies rushing to help
- NPCs trading real items at traders - artefacts for ammo, grenades, medkits
- Campfire and base behavior driven by hunger, fatigue, social needs
- Chained consequences: one event triggers another, and that triggers another - emergent behavior nobody scripted

---

Why this mod exists:

A-Life modding is hard. The engine exposes limited APIs, documentation is scarce, and most of the knowledge comes from trial and error.

Common patterns include scanning the entire world hundreds of times per frame at O(n) cost, permanently hijacking squads by overwriting engine variables like scripted_target, and heavy luabind loops that scale linearly with squad count. This causes ghosting, entity leaking, save corruption, and conflicts between mods. Engine interactions that crash are swallowed by pcall exception handlers - entities move a bit, the world feels a bit alive, but operations fail silently and degrade over time. State accumulates across saves with no cleanup. Behaviors often duplicate what the engine already handles, with no cohesive economy and no systemic purpose behind the movement.

---

The AlifePlus Approach:

Engine events are intercepted as they fire. The system reacts when something happens, not on a timer. If nothing is happening in the simulation, AlifePlus is idle.
Squads are extended, not hijacked. The engine's own job system produces the correct behavior without overwriting its variables.

Events chain deterministically. Causes dispatch consequences, and consequences change the world state, which triggers new causes. Behavior is the result of independent actors colliding in a shared simulation.
- Reactive causes: triggered by combat, wounds, deaths, item use
- Radiant causes: triggered by movement, location, and needs

AlifePlus is first and foremost a framework. Any alife scenario that can be described as "when X happens, do Y" can be implemented by registering a cause predicate and a consequence handler. The pipeline handles protection, rate limiting, tracing, squad lifecycle, and PDA routing. New causes and consequences plug in without modifying core code.

Other mods can integrate with AlifePlus at any level, from passively listening to events to registering new behaviors into the pipeline to coordinating squad control with their own alife logic. AP uses only engine-native mechanisms (scripted_target, SIMBOARD, gulag jobs) so any mod that respects the engine's own APIs will not conflict. See doc/integration-guide.md for examples, API reference, and a concrete two-mod collaboration scenario.

---

Architecture:

- Rate-limited event pipeline with token budgets, cooldowns, and separate pacing for radiant and reactive streams. Designed to prevent the ghosting and save corruption caused by squad hijacking and engine API spam.
- Protection gates at every pipeline layer: permanent NPCs, task targets, companions, and externally scripted squads are never touched. Squads owned by other mods (warfare, BAO) are excluded via an ownership registry.
- All engine interaction through xlibs, a shared library reverse-engineered from X-Ray Monolith C++ source covering squad and smart terrain APIs, event bus, synthetic callbacks, crash-safe object resolution, and microsecond profiling.
- No base script edits, no engine patches. Runtime callbacks and hooks only. Where the engine lacks a callback, a runtime hook emits a synthetic event without modifying base scripts.
- Operates above the action planner at the simulation layer, routing squads to the correct smart terrain through SIMBOARD. The engine's own gulag jobs, GOAP planner, and scheme bindings handle all behavior at the destination. AlifePlus provides the intent, the engine provides the behavior.

Performance:

- Designed performance-first, load tested (1000x load), profiled with anomaly-devtools v1.3.0
- Near-zero performance impact as seen in the screenshots
- OTel-style tracing, structured logging with rotation, microsecond profiling
- Long saves stable by avoiding permanent squad control and limiting engine pressure under load

[BENCHMARKS: two screenshots side by side]

---

Configuration:

Each cause and consequence is a module you can enable or disable through MCM. Gameplay actions (item consumption, stash looting, artefact trading, rank progression) each have their own toggle and tunable values - chances, cooldowns, thresholds, quantities, rate limits, and budgets.
Log level goes from silent to full tracing with pathing, performance timing, and PDA map markers.

Presets:
  Calm Zone: A-Life Interval 10-15, Cause Budget 5, Consequence Budget 1, Global Rate Limit 2
  Vanilla+: A-Life Interval 5, Cause Budget 10, Consequence Budget 2, Global Rate Limit 5 (defaults)
  Chaotic: A-Life Interval 1-2, Cause Budget 30+, Consequence Budget 5+, Global Rate Limit 15+, raise individual chances

---

Features:

Causes and consequences that aggregate into hundreds of combinations.

Massacre (reactive)
  - Scavenge - Nearby scavengers and predators converge on the massacre site.
  - Investigate - Victims' faction sends nearby squads to investigate.

SquadKill (reactive)
  - Revenge - Same-faction squads pursue the killer as it moves.
  - Flee - Nearby squads of the victim's faction fall back to the nearest base.

BaseKill (reactive)
  - Support - Nearby friendly squads rush to reinforce the base under attack.
  - Flee - Squads at the base evacuate to the nearest friendly position.

Elite (reactive)
  - Promote - NPCs accumulate kills to level up. 10 levels, each requiring more kills.
    Rank boost, grenades, and AP ammo scale with level.

EliteKill (reactive)
  - Bounty - Killer receives rubles scaled by the elite's level.
  - Targeted - Other elites on the same tier hunt the killer across the map.

Wounded (reactive)
  - Hunt - Predator mutants sense weakness and close in.
  - Help - Nearby same-faction squads rush to help the wounded.

Harvest (reactive)
  - Hunt - Nearby outlaws pursue whoever picked up the artefact as it moves.

Stash (radiant)
  - Loot - Stalker squad spots the stash, walks there, and loots it.
  - Ambush - Outlaw squad spots the stash and camps it. The stash is bait.
  - Fill - Stalker squad spots the stash and hides supplies inside.

Area (radiant)
  - Conquer - Squad claims the empty smart terrain. Engine spawns their faction there.

Needs (radiant)
  Stalkers have human needs. Nine Maslow-Hull drives scored by deprivation.
  The strongest unmet need wins.
  - Hunger - Finds a campfire. Eats what he is carrying - bread, sausage, canned goods etc.
  - Sleep - Waits for nightfall. Finds a campfire. Sleeps through the night.
  - Rest - Finds a campfire. Smokes a cigarette, has a drink.
  - Heal - Finds a safe location. Uses a medkit, bandage, or stimpack.
  - Shelter - Finds a safe location when exposed too long.
  - Money - Searches anomaly fields for artefacts, or hunts mutant lairs.
  - Supply - Visits a trader. Trades an artefact for AP ammo, grenades, or medical supplies.
  - Job - Guards outposts and checkpoints. Explores the Zone. Researches anomalies. Loners hunt. Monolith and Sin worship at remote sites. Military and Duty drill at forward positions.
  - Social - Finds a campfire or a safe location. Shares cigarettes and drinks.

  NPCs consume real items from their inventory on arrival. A guard smokes a cigarette on duty. A hungry stalker eats food he was carrying.
  A supply run costs some trophies, booze, an artefact and returns usables, medkits, ammunition - including AP rounds and grenades.

---

Compatibility & Safety:
- Built and tested with GAMMA, also tested with Zona and EFP
- No base script edits, no engine patches
- Works mid-save
- Story NPCs, companions, task givers, and quest squads are never touched
- Squads owned by other mods (warfare, BAO) excluded automatically via ownership registry
- No permanent squad hijacking - all scripted squads have TTL and auto-release
- Other mods can integrate at any level: listen to events, register new behaviors, or coordinate squad control
- Uses only engine-native mechanisms (scripted_target, SIMBOARD, gulag jobs) - compatible with any mod that respects the engine's own APIs
- AlifePlus does not need third-party "bridge" or "synergy" patches. The framework's own integration layer handles mod coordination natively. Mods that adopt the AP framework through the official API (see integration-guide.md) are supported. Unauthorized patches that claim compatibility are not endorsed and will cause instability, save corruption, and conflicts with both AP and the mods they claim to bridge.
- See doc/integration-guide.md for API reference and examples

---

Requirements:
- Anomaly 1.5.3
- Modded exes
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

FAQ:

How do I make events occur faster or slower?
  Calmer world: raise A-Life Interval to 8-10, lower Consequence Budget to 1, raise Radiant Cooldown to 15-20.
  Busier world: lower A-Life Interval to 2-3, raise Consequence Budget to 4, lower Radiant Cooldown to 5.
  Gung-Ho: lower A-Life Interval to 1, raise Cause and Consequence Budget to max, raise consequence chances to 50+, set Radiant Cooldown to 0. Good luck!

Does it conflict with the vanilla task system?
  No. AlifePlus validates every entity through protection gates at every pipeline layer.
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

---

Known Issues:
Map markers are a debugging feature. They lack persistence and do not update on entity death, despawn, or level transition.

---

Validation:
Multi-stage validation pipeline:
- luacheck, selene (static analysis)
- tree-sitter AST analysis, ast-grep structural patterns
- Contract rules (API safety, cross-file dependencies, cyclomatic complexity, coding standards)
- lua54 integration testing with X-Ray engine stubs
- gitleaks, trufflehog (secret scanning)
Full report in doc/test-report.log.

Credits:
Stalker_Boss - Russian translation

---

Usage and License:
- Modpacks: allowed and encouraged. Keep the readme and license files.
- Addons, patches, integrations: allowed. Credit "AlifePlus by Damian Sirbu" visibly on your mod page.
- Reproducing the implementation in other software: not allowed, even with credit.
- Full license in LICENSE file and on GitHub.
