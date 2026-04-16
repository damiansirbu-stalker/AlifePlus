AlifePlus: Emergent A-Life for STALKER Anomaly, by Damian
- Manifesto: https://github.com/damiansirbu-stalker/AlifePlus/blob/main/doc/manifesto.md
- Integration guide: https://damiansirbu-stalker.github.io/siski-report/guide.html
- Changelog: https://github.com/damiansirbu-stalker/AlifePlus/blob/main/doc/changelog
- Russian / На русском: https://github.com/damiansirbu-stalker/AlifePlus/blob/main/doc/readme_ru.txt

! Please use the RESET button in MCM when updating to a new version !

You are not special.

AlifePlus is a reactive alife framework for STALKER Anomaly.
A complete simulation layer that extends A-Life with event-driven emergent behavior, built on GSC's original AI design.
It intercepts engine events, classifies them into causes, and dispatches consequences that change the simulation.
NPCs and mutants act independently, pursue goals, react to threats, and create knock-on effects.
Squads investigate massacres, hunt artefact carriers, claim empty territory, and act on hunger, sleep, and social needs.
Everything that happens to the player happens to NPCs and mutants alike.
Every action traces back to a cause, a world state, and a mechanic. Nothing is arbitrary.

Inspired by Roadside Picnic, the original STALKER vision, and GSC's unshipped AI design documents.
The Zone runs on its own rules, and the actor is just another entity inside it.
Factions behave according to their identity.
Duty holds ground, bandits ambush and loot, ecologists research, monolith never retreats.
Two layers enforce this: alignment determines what a faction can do at all,
personality determines how likely they are to act.
See the manifesto for proof and source references.

Every action has a systemic cause. Nothing moves without a reason. The simulation runs whether you are there or not.

The mod is built around two invariants:
- Event-driven contract: nothing runs unless the engine says something happened.
- Physical simulation guarantee: nothing is spawned, teleported, or fabricated. Consequences use entities already in the simulation.

The economy uses real inventory items, and real needs drive real decisions.

---

Example scenario (systemic interaction):

- A stalker reaches a location. AlifePlus intercepts that event.
- He spots a stash ("stash" cause) and his faction's greed is high enough to loot it ("loot" consequence).
- He walks to the stash but gets ambushed by a bandit who saw it first. Bandits camp stashes ("ambush" consequence).
- The player was heading to that stash and sees the fight, so he snipes both of them.
- This mass killing ("massacre" cause) draws cowardly mutants to the bodies ("scavenge" consequence).
- The victims' faction sends squads to investigate ("investigate" consequence).
- On the way there, the scavengers get killed by loners who were out hunting ("job" consequence).
- Those loners are now tired ("needs" cause) and head back to base to smoke a cigarette and tell the story ("social" consequence).
- None of this was scripted. The player walked into something already in motion.

Example scenario (escalation chain):

- An artefact spawns. A stalker needs money ("needs" cause) and heads there to pick it up ("money" consequence).
- On the road he fights creatures and they fight back.
  A chimaera accumulates kills and becomes an alpha ("alpha promote" consequence), hitting harder, taking less, never fleeing.
- The player wounds it but it keeps fighting (panic immunity).
  He kills it and finds valuable mutant parts and an artefact in its inventory.
- Another chimaera on the same tier senses the kill and pursues the player ("alphakill targeted" consequence).
- The cycle continues, the Zone does not care.

What you'll notice:
- Squads rerouting after nearby combat deaths
- Scavengers and investigators converging on massacre sites
- Stash traffic: NPCs walking to stashes, looting, ambushing, filling
- Outlaws hunting whoever picked up an artefact
- Retaliation and retreat loops after squad wipes
- Territory flipping as stalkers and mutants claim empty outposts. Conquests decay if not held.
- Alpha mutants emerging from combat, accumulating kills, hit power buffs, and valuable loot
- Predators closing in on wounded targets, allies rushing to help
- NPCs trading real items at traders - artefacts for ammo, grenades, medkits
- Campfire and base behavior driven by hunger, fatigue, social needs
- Day/night cycle: stalkers work and trade by day, rest and sleep at campfires by night.
- Nocturnal predators hunt at night and retreat to lairs at dawn. Diurnal mutants do the opposite.
- Chained consequences: one event triggers another, and that triggers another - emergent behavior nobody scripted
- Bandits looting and ambushing while Duty patrols and holds the line
- Loners chasing artefacts while Military stays close to base
- Monolith pushing into territory while Renegades scatter at first contact
- Factions responding to the same event based on personality, not random chance

---

Why this mod exists:

A-Life modding is hard. The engine exposes limited APIs, documentation is scarce, and most knowledge comes from trial and error.

Common patterns in A-Life modding scan the world every frame, permanently hijack squads
by overwriting engine variables, swallow crashes silently, and accumulate stale state
across saves with no cleanup. The result is poor performance, entity leaking, ghosting,
save corruption, and mod conflicts that get worse the longer you play.

---

The AlifePlus Approach:

Engine events are intercepted as they fire. The system reacts when something happens, not on a timer.
If nothing is happening in the simulation, AlifePlus is idle.
Squads are extended, not hijacked. The engine's own job system produces the correct behavior without overwriting its variables.

Events chain deterministically. Causes dispatch consequences, consequences change the world state, which triggers new causes.
Behavior is the result of independent actors colliding in a shared simulation.
- Reactive causes: triggered by combat, wounds, deaths, item use
- Radiant causes: triggered by movement, location, and needs

Every action is bounded by awareness. Three range tiers govern how far a squad searches:
- EyeRange: what the squad can see. Opportunities like stashes and unclaimed territory.
- SignalRange: PDA radio. Stalkers hear about massacres, base attacks, trader locations.
- ScentRange: scent tracking. Mutants smell corpses, track wounded prey, follow trails.
Range tiers are grounded in GSC's original AI design documents (PersonalEyeRange, EnemyDetectProbability).
EyeRange validated empirically: nearest-neighbor distance measured in-engine across 14 levels
(Escape, Garbage, Bar, Darkvalley, Military, Yantar, Deadcity, Red Forest, Radar, Pripyat,
Marsh, Trucks Cemetery, Promzona, Pole). Median NN distance 44-148m, p90 66-188m.

AlifePlus is first and foremost a framework.
Any alife scenario that can be described as "when X happens, do Y" can be implemented by registering a cause and a consequence.
The pipeline handles protection, rate limiting, tracing, squad lifecycle, and PDA routing.
New causes and consequences plug in without modifying core code.

Other mods can integrate at any level: passively listen to events, register new behaviors, or coordinate squad control.
AP uses only engine-native mechanisms (scripted_target, SIMBOARD, gulag jobs).
Any mod that respects the engine's own APIs will not conflict.
See guide.html on the project site for examples, API reference, and a concrete two-mod collaboration scenario.

---

Architecture:

- Rate-limited event pipeline with token budgets, cooldowns, and separate pacing for radiant and reactive streams.
- Protection gates at every pipeline layer: permanent NPCs, task targets, companions, externally scripted squads.
  Squads owned by other mods (warfare, BAO) are excluded via an ownership registry.
- AlifePlus runs on xlibs, a reverse-engineered API that wraps the X-Ray engine source.
  It was built, load-tested, and validated against the C++ implementation.
  Core patterns were studied from the most accomplished mods in the Anomaly ecosystem.
- No base script edits, no engine patches. Runtime callbacks and hooks only.
- Operates above the action planner at the simulation layer, routing squads through SIMBOARD.
  The engine's own gulag jobs, GOAP planner, and scheme bindings handle all behavior at the destination.

Performance:

- Performance was the first design constraint. The pipeline was profiled with anomaly-devtools v1.3.0 and load-tested at 1000x simulated pressure.
- Near-zero performance impact as seen in the screenshots
- OTel-style tracing, structured logging with rotation, microsecond profiling
- Long saves stable by avoiding permanent squad control and limiting engine pressure under load

[BENCHMARKS: two screenshots side by side]

---

Configuration:

Each cause and consequence is a module you can enable or disable through MCM.
Gameplay actions have their own toggles and tunable values: chances, cooldowns, thresholds, rate limits, and budgets.
Log level goes from silent to full tracing with pathing, performance timing, and PDA map markers.

Presets:
  Calm Zone: A-Life Interval 15-20, Cause Budget 5, Consequence Budget 1, Global Rate Limit 2
  Vanilla+: A-Life Interval 12, Cause Budget 10, Consequence Budget 2, Global Rate Limit 5 (defaults)
  Chaotic: A-Life Interval 2-4, Cause Budget 30+, Consequence Budget 5+, Global Rate Limit 15+

---

Features:

Causes and consequences that aggregate into hundreds of combinations.

Reactions

  World events trigger responses. A death, a healing, a pickup. Consequences search
  within signal or scent range and evaluate alignment and personality.

  Massacre
    - Scavenge - Nearby scavengers and predators converge on the massacre site.
    - Investigate - Victims' faction sends nearby squads to investigate.

  SquadKill
    - Revenge - Same-faction squads pursue the killer as it moves.
    - Flee - Nearby squads of the victim's faction fall back to the nearest base.

  BaseKill
    - Support - Nearby friendly squads rush to reinforce the base under attack.
    - Flee - Squads at the base evacuate to the nearest friendly position.

  Alpha
    - Promote - Mutants accumulate kills to level up. 10 levels. Hit power bonus/reduction
      scales with level, panic immunity, valuable loot injected into inventory.

  AlphaKill
    - Targeted - Other alphas on the same level hunt the killer.

  Wounded
    - Hunt - Predator mutants sense weakness and close in.
    - Help - Nearby same-faction squads rush to help the wounded.

  Harvest
    - Rob - Nearby outlaws pursue whoever picked up the artefact.
    - Haunt - Aberrant mutants converge on the artefact pickup site.

Opportunities

  Squad sees what's nearby and acts. Consequences search within eye range and
  evaluate alignment and personality. Per-squad drive cooldown prevents repeated firing.

  Stash
    - Loot - Stalker squad spots the stash, walks there, and loots it.
    - Ambush - Outlaw squad spots the stash and camps it. The stash is bait.
    - Fill - Stalker squad spots the stash and hides supplies inside.

  Area
    - Conquer - Organized and aggressive factions claim empty territory. Ecologists, loners, and renegades never conquer.
      Conquered territory decays over time if not held (decay hours configurable in MCM).

Needs

  Stalkers have human needs. Nine drives inspired by Maslow-Hull, scored by how long
  since last fulfilled. The most deprived need wins. Consequences search within signal
  range and evaluate alignment and personality.
  - Hunger - Finds a campfire. Eats what he is carrying - bread, sausage, canned goods etc.
  - Sleep - Finds a campfire during dormant hours. Sleeps.
  - Rest - Finds a campfire. Smokes a cigarette, has a drink.
  - Heal - Finds a safe location. Uses a medkit, bandage, or stimpack.
  - Shelter - Finds a safe location when exposed too long.
  - Money - Searches anomaly fields for artefacts, or hunts mutant lairs.
  - Supply - Visits a trader. Trades an artefact for AP ammo, grenades, or medical supplies.
  - Job - Guards outposts and checkpoints. Explores the Zone. Researches anomalies.
  - Social - Finds a campfire or a safe location. Shares cigarettes and drinks.

  NPCs consume real items from their inventory on arrival. A guard smokes a cigarette on duty.
  Stalkers go to traders to swap artefacts for ammo, grenades, or medical supplies.

Instincts

  Mutants have instincts. Four drives scored by deprivation, same as stalker needs.
  The strongest unmet instinct wins. Consequences search within scent range and
  evaluate alignment and personality.
  - Feed - Mutants move to open territory during active hours. Predators and prey meet on shared hunting grounds.
  - Sleep - Mutants return to rest locations during dormant hours.
    Cowardly sleep in fields, feral den in lairs, predators use lairs or buildings, aberrant shelter underground.
  - Explore - Mutants wander to a different territory or lair during active hours.
  - Socialize - Pack animals move toward smart terrains where same-faction squads are present.

Day/Night Cycle
  Stalkers and mutants follow a day/night activity cycle.
  During active hours, creatures feed, explore, work, and trade.
  During dormant hours, they seek shelter and sleep.
  Stalkers are active during the day and sleep at night.
  Nocturnal mutants -- bloodsuckers, lurkers, chimeras, zombies, fractures --
  are active at night and sleep during the day. All other species are diurnal.
  The same mechanism gates both stalker needs and mutant instincts.

Alignment
  GSC classified characters as principled, self-serving, or unprincipled.
  AlifePlus maps this to factions as a hard gate. Military never flees.
  Ecologists never conquer. Renegades never investigate. Outlaws never help the wounded. These are structural, not tunable.
  Mutant species operate on two independent axes.
  Behavioral: cowardly (flesh, zombie, rats), feral (dogs, boars, snork, gigant),
  predator (bloodsucker, chimera, lurker), aberrant (controller, burer, poltergeist).
  Activity: nocturnal (bloodsucker, lurker, chimera, zombie, fracture) vs diurnal.
  Behavioral determines what a species does. Activity determines when.

Personality
  Every stalker faction and mutant species has a personality profile derived from GSC's
  original AI design documents. Traits include aggression, greed, survival, perception,
  territory, relation, and discipline. Each consequence checks at most two relevant
  traits, averages them, and rolls against the result clamped to a global floor and
  ceiling (MCM personality_min / personality_max). Both stalkers and mutants share the
  same system. Trait values are direct probabilities grounded in GSC lore.

---

Compatibility & Safety:
- Compatible with all modded exe variants (Demonized, AOE, MT)
- Built and tested with GAMMA, also tested with Zona and EFP
- No base script edits, no engine patches
- Works mid-save
- Story NPCs, companions, task givers, and quest squads are never touched
- Squads owned by other mods (warfare, BAO) excluded automatically via ownership registry
- No permanent squad hijacking - all scripted squads have TTL and auto-release
- Other mods can integrate at any level: listen to events, register new behaviors, or coordinate squad control
- Uses only engine-native mechanisms (scripted_target, SIMBOARD, gulag jobs).
- AlifePlus does not need third-party "bridge" or "synergy" patches.
  Mods that adopt the AP framework through the official API (see guide.html on the project site) are supported.
  Unauthorized patches that claim compatibility are not endorsed and will cause instability and save corruption.
- See guide.html on the project site for API reference and examples

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
  Gung-Ho: lower A-Life Interval to 1, raise Cause and Consequence Budget to max, set Radiant Cooldown to 0. Good luck!

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
Altogolik - support, ideas, source materials

---

Usage and License:
- Modpacks: allowed and encouraged. Keep the readme and license files.
- Addons, patches, integrations: allowed. Credit "AlifePlus by Damian Sirbu" visibly on your mod page.
- Reproducing the implementation in other software: not allowed, even with credit.
- Full license in LICENSE file and on GitHub.
