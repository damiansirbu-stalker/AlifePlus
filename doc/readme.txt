AlifePlus: Emergent A-Life for STALKER Anomaly, by Damian
Latest: 1.2.0 (requires xlibs >= 1.2.2)
GitHub: https://github.com/damiansirbu-stalker/AlifePlus

! Please use the RESET button in MCM when updating to a new version !

You are not special.

AlifePlus is a behavior mod inspired by Roadside Picnic and the original STALKER vision. GSC wanted a simulation where the game world operates independently of player actions.
I respected that and took it further, in AlifePlus the simulation is fair and equal for all creatures on the map, and the player is just one entity.

Why use AlifePlus? You can decide by comparing how this mod works to any other mod.

Classical:
Most A-Life mods scan the entire game world on a timer, entity by entity and smart by smart. O(n) per tick, ticking every frame.
They pick squads and override their simulation with scripted behavior like a permanent hijack, overwriting important variables used by the engine or other mods.
Heavy loops with API calls cause ghosting, leaking, or save corruption.
Behaviors like trading have no cohesive economy. Movement is just for show, performing things the engine already handles or that dedicated mods do better.
The world feels more alive because things move around, but the movement has no meaning and the performance cost adds up.

AlifePlus:
Engine events are intercepted as they fire. The system reacts when something happens, not on a timer.
Squads are extended, not hijacked. The engine's own job system produces the correct behavior without overwriting its variables.

All engine interaction goes through xlibs, a shared library reverse-engineered from X-Ray Monolith C++ source covering squad and smart terrain APIs, event bus, synthetic callbacks, crash-safe object resolution, and microsecond profiling.
Where the engine lacks a callback, a runtime hook emits a synthetic event without modifying base scripts.
Code patterns validated against Demonized, Vintar0, RavenAscendant, xcvb.

Rate-limited event pipeline with token budgets, designed to avoid ghosting and save corruption caused by squad hijacking and engine API spam.

Events chain into causes, causes dispatch consequences, and consequences become new causes. Emergent gameplay that nobody scripted.
Causes are either reactive (triggered by combat, wounds, deaths) or radiant (triggered by movement and location), all event-driven.

The economy uses real inventory items and needs drive real decisions. Performance cost is near zero.

Example scenario:
A stalker reaches a location. AlifePlus intercepts that event.
He spots a stash ("stash" cause) and decides to loot it ("loot" consequence), not knowing if anything is inside.
He walks the whole distance to the stash but gets ambushed by another stalker who saw it first and decided to rob people ("ambush" consequence).
The player was heading to that stash and sees the fight, so he snipes both of them.
This mass killing ("massacre" cause) draws scavengers to the bodies ("scavenge" consequence) and the victims' faction sends squads to investigate ("investigate" consequence).
On the way there, the scavengers get killed by loners who were out hunting ("job" consequence).
Those loners are now tired ("needs" cause) and head back to base to smoke a cigarette and tell the story ("social" consequence).
None of this was scripted. The player walked into something already in motion.

Another example:
An artefact spawns. A stalker needs money ("needs" cause) and heads there to pick it up ("money" consequence).
On the road he fights creatures and gains enough kills to become an elite ("elite promote" consequence).
Now stronger but wounded, he bleeds ("wounded" cause) and a chimaera senses weakness and chases him down ("wounded hunt" consequence).
The chimaera kills him and gains enough kills to become an elite itself ("elite promote" consequence), a real problem in the area.
If the player kills it, he collects a bounty ("elite bounty" rewards the player). If an NPC kills it, the NPC may become an elite ("elite bounty" also rewards NPCs).
The cycle continues, the Zone does not care.

The mod was designed performance-first, load tested with my own testing framework (1000x load) and profiled with anomaly-devtools v1.3.0. The performance impact is almost inexistent as seen in the screenshots.
The entire pipeline is instrumented with OTel-style tracing, structured logging with rotation, and microsecond profiling from the engine's own timer, so every operation is measurable.

AlifePlus operates above the action planner at the simulation layer, routing squads to the correct smart terrain through SIMBOARD.
The engine's own gulag jobs, GOAP planner, and scheme bindings handle all behavior at the destination. It already knows how to make guards patrol, sleepers sleep, collectors collect. AlifePlus provides the intent, the engine provides the behavior.

[BENCHMARKS: two screenshots side by side]

Each cause and consequence is a module you can enable or disable through MCM, and within each module you can tune chances, cooldowns, thresholds, rate limits, and budgets individually.
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
  Heal - Returns to a friendly base. Uses a medkit, bandage, or stimpack.
  Shelter - Returns to a friendly base when exposed too long.
  Money - Searches anomaly fields for artefacts, or hunts mutant lairs.
  Supply - Visits a trader. Trades an artefact for AP ammo, grenades, or medical supplies.
  Job - Guards their base. Explores the Zone. Researches anomalies. Loners hunt. Monolith and Sin worship. Military and Duty drill.
  Social - Finds a campfire or returns to base. Shares cigarettes and drinks.

  NPCs consume real items from their inventory on arrival. A guard smokes a cigarette on duty. A hungry stalker eats food he was carrying. 
  A supply run costs some trophies, booze, an artefact and returns usables, medkits, ammunition - including AP rounds and grenades.

Requirements:
Anomaly 1.5.3
Modded exes
xlibs 1.2.2 (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
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
    Calmer world: raise A-Life Interval to 8-10, lower Consequence Budget to 1.
    Busier world: lower A-Life Interval to 2-3, raise Consequence Budget to 4.
	Gung-Ho: lower A-Life Interval to 1, raise Cause Budget and per Consequence to max, raise consequence chances to 50+, lower PDA chances to 10. Good luck!

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
Map markers are a debugging feature (log_level=DEBUG). They lack persistence and do not update on entity death, despawn, or level transition.
PDA system still needs some work, messages and their sources are not what they could be yet

Development:
Written against X-Ray Monolith engine source, Demonized exes source code, and Anomaly 1.5.3 unpacked gamedata.
Validated by a custom multi-stage pipeline: luacheck, selene, tree-sitter AST analysis, ast-grep structural patterns,
contract rules (API safety, cross-file dependencies, cyclomatic complexity, coding standards),
lua54 integration testing with X-Ray engine stubs, gitleaks and trufflehog secret scanning.
Full report in doc/test-report.log.

Credits:
Stalker_Boss - Russian translation

Versions:

  1.2.0
    Configurability:
      Added: per-consequence walk/rush, tactical defaults rush, routine defaults walk (MCM on/off)
      Added: stash protection, NPCs skip stashes with essential items (MCM on/off)
      Added: base guard for stash, loot/fill/ambush skip when squad already at base
      Changed: revenge and elite hunters skip kills that don't affect faction relations
    Smart terrain routing:
      Added: capacity-aware routing, all paths check real population + in-transit count
      Added: on-arrival redirect, squads at full smarts complete handler then relocate
      Changed: territory conquest extracted to own module with independent save/load
    NPC protection:
      Added: unscriptable guard at cause, consequence, and tracker levels
      Added: is_externally_scripted check (Warfare, condlist, random patrol compatibility)
      Fixed: Aslan (Dead City barman) not in unscriptable list due to typo (xlibs)
    Performance:
      Changed: debug overhead eliminated when off, simplified internals
    Compatibility:
      Changed: semver dependency gate, xlibs patch bumps no longer cascade AP releases
      Changed: squad TTL uses game time instead of wall clock, survives sleep acceleration
    Observability:
      Added: structured arrival logging for needs consequences (destination, action, items)
      Added: diagnostic logging for smart search and squad filtering in debug mode
    Fixed:
      Fixed: elite kill PDA always using mutant template (is_stalker not persisted)
      Fixed: duplicate consequence handlers accumulating across save/load cycles
      Fixed: destination capacity only covered needs, all other paths bypassed it
      Fixed: artefact drop/pickup cycling triggering infinite harvest chasers (per-section cooldown)
    Removed:
      Removed: dead enum values

  1.2.0-RC1 - "Needs"
    Needs:
      Added: 9 new causes/needs: hunger, sleep, rest, heal, shelter, money, supply, job, social
      Added: 15 new consequences with per-need MCM controls (enable, threshold, chance, action)
      Added: Night gate, faction-aware destination filter, destination limiter
      Added: 8 arrival actions: NPCs consume real inventory items or trade artefacts for supplies
      Added: Needs persisted per-squad in save data
    Event pipeline:
      Added: Radiant/reactive split with independent token buckets
      Added: Consequence token bucket replaces global cause lock
      Added: Radiant exhaustion peek, actor_on_interaction adapter, unscriptable guard
    PDA:
      Added: 168 message variants across 40 categories, stalker/mutant distinction
    MCM:
      Added: Reset button, retuned default chances, cleaned up labels
      Added: Russian translation (Stalker_Boss)
    Performance:
      Changed: MCM snapshot pattern, debug/trace function-swap (zero cost at WARN)
    Docs:
      Added: manifesto.md with GSC vision, engine source proof, developer quotes
    Fixed:
      Fixed: Reactive event starvation (cause lock starving needs and area causes)
      Fixed: wounded_help faction matching (actor_stalker prefix)
      Fixed: PDA messages now distinguish stalkers from mutants
      Fixed: supply_trader exchange giving free items (now requires artefact)

  1.1.1
    Fixed: dependency gate uses exact version match instead of string comparison

  1.1.0
    10-level elite system replaces binary elite/legend. Alife Ratio balances simulation across maps.
    Elite: 10 levels, AP ammo restock (best k_ap for weapon), engine rank boost (up to 20% accuracy), grenade restock. Alife Ratio: Bresenham ratio gate for on/off-map event distribution, MCM slider (-10 to +10).
    Fixed: elitekill_targeted never firing (bounty was stopping the chain)
    Fixed: PDA message tone, elite buff performance

  1.0.1
    Dependency gate, bugfixes, Russian translation.
    Added: dependency gate - clear error message if xlibs is missing or outdated
    Added: Russian translation (Stalker_Boss)
    Fixed: companion squads excluded from squad searches (xlibs)
    Fixed: player-created stashes excluded from stash searches (xlibs)

  1.0.0
    First release. Event-driven emergent A-Life system.
    10 causes, 17 consequences. Full MCM configuration.
    Added: reactive causes (massacre, squadkill, basekill, elite, legend,
           legendkill, wounded, harvest)
    Added: radiant causes (stash, area)
    Added: chase API, area conquest, hit modifiers, rank boost
