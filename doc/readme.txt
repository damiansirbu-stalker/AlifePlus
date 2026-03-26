AlifePlus: Emergent A-Life for STALKER Anomaly, by Damian
Latest: 1.2.0-RC1 (xlibs 1.2.0)

You are not special.

AlifePlus is a behavior mod that follows Roadside Picnic and the original STALKER, not "warfare" or "action" alternatives.

GSC described their vision: "We wanted a simulation where the game world operates independently of player actions."
I respected this and went further. In AlifePlus the simulation runs independent of the player, fair and equal for all creatures on the map, where the player is just one entity.

Why use AlifePlus? What makes it different?
You can decide by comparing how my mod works to any other mod, let's call it "classical".

Classical:
- scan the entire game world on a timer, entity by entity and smart by smart (O(n) per tick, ticking every frame)
- pick squads and override their simulation with scripted behavior (permanent hijack)
- run heavy loops with API calls that can cause ghosting, leaking, or save corruption (unguarded side effects)
- add features like trading without an economy, moving around just for show, jobs or blowout reactions the engine already handles, patrols or night mutants that dedicated mods do better (redundant)
- the world feels more alive because things move around, but the movement has no cause and the performance cost adds up (no design)

AlifePlus:
- Every xray event is traced and listened to. Each has dedicated rate limiters that ensure there is no chance of performance degradation, then:

Example scenario:
- Let's say a stalker walks around. I intercept that event
- The stalker radiantly looks around and sees a stash. He decides to loot it but he does not know if there is something inside
- He walks the whole distance to the stash but when he gets there, he is ambushed by another stalker that also saw the stash and decided to ambush and rob people
- The player was heading to that stash and sees the fight, so he snipes both of them
- But this triggers a massacre cause, which may lead to scavengers heading to the location, or vice versa, the victim's faction going there to investigate
- But on the way to the investigation site, a new event happens... and the cycle continues

Another example:
- Let's say an artefact spawns
- A hungry stalker goes there to pick it up and make a living
- On the road, he gets to fight some creatures and gains enough status to become an elite
- Now he is stronger but wounded, he bleeds
- A chimaera smells the blood and kills him
- Now the chimaera is an elite and becomes a real problem in the area
- Stalkers or the player kill that chimaera
- If the player kills it, he collects a bounty. If an NPC kills it, the NPC may become an elite... and the cycle continues

AlifePlus was built from the X-Ray engine C++ source, every function extracted, documented with full API and measured in real time.
Where the engine lacks a callback, a runtime hook emits a synthetic event without modifying base scripts.
All engine work is extracted into xlibs: source-inspected APIs for squad search, smart terrain filters, event bus, chase lifecycle, PDA dispatch, and safe server object resolution.

Details of the system can be found inside architecture.md, but in a nutshell:
- event-driven reactive architecture, zero cost when idle
- nothing is scripted, nothing is iterated, forced or hijacked
- no fake spawns, no teleporting, no entity creation
- original xray events are intercepted, evaluated and aggregated into "causes"
- causes can be reactive, a direct reaction to an event
- causes can also be radiant, where the NPC autonomously evaluates its surroundings
- causes are decoupled from, and lead to consequences
- causes and consequences chain nondeterministically, leading to emergent gameplay

Zero performance impact. The attached screenshot shows zero performance hit with 1000x load, measured for almost 8 hours continuously. Event-driven: zero cost when nothing happens.
Instead of overriding the engine's job planner, AlifePlus operates at the simulation layer: route the squad to the correct smart terrain based on evaluated need, and the engine's gulag system produces the correct behavior by design.

Nothing is faked and every reaction is earned from world state. Every creature, item or object involved was already there, living its own A-Life, and AlifePlus just extends what it is already doing.
The system works with original xray events, moves real entities across real distances, and unfolds consequences without forcing, spawning, or teleporting

Every feature traces back to GSC's original A-Life vision or documented developer intent. The full design rationale with engine source evidence and developer quotes is in manifesto.md.

Every cause and consequence is individually toggleable through MCM.
Chances, cooldowns, thresholds, and rate limits are all configurable.
Log level goes from silent to full tracing and pathing, performance timing, and PDA map markers on every operation.

Features:

10 causes, 31 consequences. Causes detect patterns in the world. Consequences act on them.
They chain -- a consequence creates the conditions for the next cause.
Reactive causes fire on world events: a death, a medkit used, an artefact picked up.
Radiant causes fire when a squad evaluates its surroundings on smart terrain transition.

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
  Promote - NPC accumulated enough kills to level up. 10 levels, each requiring more kills.
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
  Loot - Discovering stalker squad walks to the stash and loots it.
  Ambush - Discovering outlaw squad camps at the stash. The stash is bait.
  Fill - Discovering stalker squad walks to the stash and hides supplies inside.

Area (radiant)
  Conquer - Squad claims the empty smart terrain. Engine spawns their faction there.

Needs (radiant)
  Stalkers have human needs. Nine Maslow-Hull drives scored by deprivation.
  The strongest unmet need wins.

  Hunger - Finds a campfire. Eats what he is carrying -- bread, sausage, canned goods.
  Sleep - Waits for nightfall. Finds a campfire. Sleeps through the night.
  Rest - Finds a campfire. Smokes a cigarette, has a drink.
  Heal - Returns to a friendly base. Uses a medkit, bandage, or stimpack.
  Shelter - Returns to a friendly base when exposed too long.
  Money - Searches anomaly fields for artefacts, or hunts mutant lairs.
  Supply - Visits a trader. Trades an artefact for AP ammo, grenades, or medical supplies.
  Job - Guards their base. Explores the Zone. Researches anomalies.
        Monolith and Greh worship. Military and Duty drill.
  Social - Finds a campfire or returns to base. Shares cigarettes and drinks.

  NPCs consume real items from their inventory on arrival. A guard smokes a cigarette on duty. A hungry stalker eats food he was carrying. A supply run costs an artefact and returns ammunition -- including AP rounds and grenades. If the NPC has nothing to consume, the need is still addressed -- he walked there and the drive resets.

FAQ:

Does it work with GAMMA?
  Built and tested with GAMMA.
  Works directly with X-Ray engine callbacks without modifying base scripts.

Does it work with other A-Life mods?
  No known incompatibilities, including with warfare and AI addons.

Does it work mid-save?
  Yes.

Can I configure everything?
  Every cause and consequence is individually toggleable through MCM.
  Chances, cooldowns, thresholds, and rate limits are all adjustable.
  The defaults are tuned for a Roadside Picnic experience.
  Adjust after playing, not before.

Do I need offline combat enabled?
  No. AlifePlus never requires or prefers online simulation.
  It works with all offline settings, including fully disabled.
  Offline combat produces fewer combat events but the system generates
  other events regardless, so A-Life activity continues.

Known issues:
Map markers are a debugging feature (log_level=DEBUG).
They lack persistence and do not update on entity death, despawn, or level transition.

Requirements:
Anomaly 1.5.3
Modded exes
xlibs 1.2.0 (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
MCM

Install (MO2):
1. Install xlibs
2. Install AlifePlus
3. Load order does not matter
4. Configure via MCM

Uninstall (MO2):
Disable or remove in MO2.

Credits:
Stalker_Boss - Russian translation

Development:
Source: https://github.com/damiansirbu-stalker/AlifePlus
Releases: https://github.com/damiansirbu-stalker/AlifePlus/releases
Written against X-Ray Monolith engine source, Demonized exes source code, and Anomaly 1.5.3 unpacked gamedata.
Code patterns and engine usage validated against established work by reputable GAMMA modders (Demonized, Vintar0, RavenAscendant, xcvb).
The code is validated in real time by a multi-stage pipeline: luacheck, selene, tree-sitter AST analysis, contract rules, cross-file dependency resolution, cyclomatic complexity analysis, crash and vulnerability pattern detection, lua54 integration testing with X-Ray engine stubs, gitleaks secret scanning.
Full report in doc/test-report.log.

License:
MIT License. See LICENSE file.

Versions:

  1.2.0-RC1 -- "Needs"
    Stalkers have human needs. Nine Maslow-Hull drives produce emergent daily routines.
    A hungry stalker finds a campfire and eats. A tired one sleeps through the night. Guards patrol their base. Loners hunt anomalies for money. Scientists research. Monolith prays. Nothing scripted, nothing faked.
    Needs: 9 drives, 15 consequences, per-need MCM (enabled, threshold, chance), night gate for sleep, faction-aware destinations, destination limiter. 8 arrival actions where NPCs consume real items from inventory (food, cigarettes, drinks, medkits) or trade artefacts for AP ammo, grenades, and supplies.
    Pipeline: split radiant/reactive event limiters, consequence token bucket, actor_on_interaction adapter for idle on-map squads. Reactive events (deaths, medkits, artefact pickups) no longer starved by radiant evaluation. Player-observed events increase significantly.
    Removed: global cause lock, ap_stats, xgoap_anim (vanilla gulag handles animations)
    Fixed: reactive event starvation (100% of death/medkit/item events blocked)
    Fixed: cause lock blocking 90% of predicate passes, starving needs and area causes
    Fixed: wounded_help faction matching (actor_stalker prefix stripping)
    Fixed: elite PDA messages now distinguish stalkers from mutants
    Fixed: supply_trader exchange giving free items (now requires artefact)
    MCM: full reset button in General tab (resets all AlifePlus settings across every tab and sub-tab)
    MCM: cleaned up labels and descriptions (units moved to tooltip, gate numbers corrected)

  1.1.1
    Fixed: dependency gate uses exact version match instead of string comparison

  1.1.0
    10-level elite system replaces binary elite/legend. Going Speed balances simulation across maps.
    Elite: 10 levels, AP ammo restock (best k_ap for weapon), engine rank boost (up to 20% accuracy), grenade restock. Going Speed: Bresenham ratio gate for on/off-map event distribution, MCM slider (-10 to +10).
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
