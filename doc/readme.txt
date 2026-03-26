AlifePlus: Emergent A-Life for STALKER Anomaly, by Damian
Latest: 1.2.0-RC1 (xlibs 1.2.0)
GitHub: https://github.com/damiansirbu-stalker/AlifePlus
Aside from usual architecture.md the repo contains manifesto.md, which tries to map AlifePlus design and features to GSC's original vision with source engine proof and quotes from developers

You are not special.

AlifePlus is a behavior mod that follows Roadside Picnic and the original STALKER, not "warfare" or "action" alternatives.

GSC described their vision: "We wanted a simulation where the game world operates independently of player actions."
I respected this and went further. In AlifePlus the simulation runs independent of the player, fair and equal for all creatures on the map, where the player is just one entity.

Why use AlifePlus? What makes it different?
You can decide by comparing how this mod works to any other mod, let's call it "classical".

Classical:
- scan the entire game world on a timer, entity by entity and smart by smart (O(n) per tick, ticking every frame)
- pick squads and override their simulation with scripted behavior like a permanent hijack, overwriting important variables used by the engine or other mods
- run heavy loops with API calls that can cause ghosting, leaking, or save corruption 
- add behaviors like trading without a cohesive economy, moving around just for show, perform behaviors the engine already handles, or that dedicated mods do better 
- the world feels more alive because things move around, but the movement has no meaning and the performance cost adds up

AlifePlus:
- xray events are traced and listened to. There are currently 5 layers of rate limiters and filters ensuring a monotone and lightweight stream of events.

Example scenario:
- Let's say a stalker walks around. AlifePlus intercepts that event
- The stalker radiantly looks around and sees a stash ("stash seen" cause). He decides to loot it ("loot" consequence) but he does not know if there is something inside.
- He walks the whole distance to the stash but when he gets there, he is ambushed by another stalker that also saw the stash and decided to ambush and rob people ("ambush" consequence).
- The player was heading to that stash and sees the fight, so he snipes both of them
- But this mass killing ("massacre" cause) may lead to scavengers heading to the location ("scavangers" consequence), or vice versa, the victim's faction going there to investigate ("faction investigate" consequence)
- But while heading to scavange the site, the mutants end up getting killed by some loners that were out hunting ("needs" cause, "job" consequence)
- Those loners are now tired ("needs" cause) and go back to base to smoke a cigarette and tell the story ("social" consequence)
- The cycle continues

Another example:
- Let's say an artefact spawns
- A stalker needed money ("needs" cause) and goes there to pick it up and make a living ("money" consequence)
- On the road, he gets to fight some creatures and gains enough status to become an elite ("elite promote" consequence)
- Now he is stronger but wounded, he bleeds ("wounded" cause) and this causes a chimaera to chase ("wounded hunt" consequence) and kill him 
- Now the chimaera also got enough kills to be an elite ("elite promote" consequence) and becomes a real problem in the area
- Stalkers or the player kill that chimaera
- If the player kills it, he collects a bounty ("elite bounty" rewards the layer). If an NPC kills it, the NPC may become an elite ("elite bounty" also rewards NPCs)
- The cycle continues, the Zone does not care

AlifePlus was built from the X-Ray engine C++ source, with functions extracted, documented with full API and measured in real time. Where the engine lacks a callback, a runtime hook emits a synthetic event without modifying base scripts.
All engine work is extracted into xlibs: source-inspected APIs for squad search, smart terrain filters, event bus, chase lifecycle, PDA dispatch, and safe server object resolution.

Details of the system can be found inside architecture.md, but in a nutshell:
- event-driven reactive architecture, zero cost when idle
- tries to be "purist" and loyal to CSG original vision
- nothing is scripted, nothing is iterated, forced or hijacked
- no NPC spawns, no teleporting, no shenanigans
- original xray events are intercepted, evaluated and aggregated into "causes"
- causes can be reactive, a direct reaction to an event
- causes can also be radiant, where the NPC autonomously evaluates its surroundings
- causes are decoupled from, and lead to consequences
- causes and consequences chain nondeterministically, leading to emergent gameplay

The mod was designed performance-first. Load tested with my own testing framework (1000x load) and profiled with anomaly-devtools v1.3.0. The performance impact is almost inexistent as seen in the screenshots.
Instead of overriding the engine's job planner, AlifePlus operates at the simulation layer: route the squad to the correct smart terrain based on evaluated need, and the engine's gulag system produces the correct behavior by design.

Nothing is faked and every reaction is earned from world state. Every creature, item or object involved was already there, living its own A-Life, and AlifePlus just extends what it is already doing.
The system works with original xray events, moves real entities across real distances, and unfolds consequences without forcing, spawning, or teleporting

Every cause and consequence is individually toggleable through MCM.
Chances, cooldowns, thresholds, and rate limits are all configurable.
Log level goes from silent to full tracing and pathing, performance timing, and PDA map markers on every operation.

Features: 10 causes, 31 consequences (1.2.0 RC1) that can aggregate into hundreds of combinations

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
xlibs 1.2.0 (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
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
    Calmer world: raise A-Life Distributor to 8-10, lower Budget per Consequence to 1.
    Busier world: lower A-Life Distributor to 2-3, raise Budget per Consequence to 4.
	Gung-Ho: lower A-Life Distributor to min, Budget per Cause and per Consequence to max, raise consequence chances to 50+, lower PDA chances to 10. Good luck!

Does it work with GAMMA?
  Built and tested with GAMMA, also tested partially with Zona and EFP.
  Works directly with X-Ray engine callbacks without modifying base scripts.

Does it work with other A-Life mods?
  No known incompatibilities, including with warfare and AI addons, but will conflict behavior-wise with mods doing a lot of scripting on squads.

Does it work mid-save?
  Yes.

Can I configure it?
  Everything is individually toggleable through MCM.
  Chances, cooldowns, thresholds, and rate limits are all adjustable.
  The defaults are tuned for a Roadside Picnic experience.
  Adjust after playing, not before.

Do I need offline combat enabled?
  No. AlifePlus never requires or prefers online simulation.
  It works with all offline settings, including fully disabled.
  Offline combat produces fewer combat events but the system generates
  other events regardless, so A-Life activity continues.

Known Issues:
Map markers are a debugging feature (log_level=DEBUG). They lack persistence and do not update on entity death, despawn, or level transition.
PDA system still needs some love, messages and their sources are not what they could be yet

Development:
Written against X-Ray Monolith engine source, Demonized exes source code, and Anomaly 1.5.3 unpacked gamedata.
Code patterns and engine usage validated against established work by reputable GAMMA modders (Demonized, Vintar0, RavenAscendant, xcvb).
The code is validated in real time by a multi-stage pipeline: luacheck, selene, tree-sitter AST analysis, contract rules, cross-file dependency resolution, cyclomatic complexity analysis, crash and vulnerability pattern detection, lua54 integration testing with X-Ray engine stubs, gitleaks secret scanning.
Full report in doc/test-report.log.

Credits:
Stalker_Boss - Russian translation

Versions:

  1.2.0-RC1 - "Needs"
    Stalkers have human needs. Nine Maslow-Hull drives produce emergent daily routines.
    A hungry stalker finds a campfire and eats. A tired one sleeps through the night. Guards patrol their base. Loners hunt anomalies for money. Scientists research. Monolith prays. Nothing scripted, nothing faked.
    Needs: 9 drives, 15 consequences, per-need MCM (enabled, threshold, chance), night gate for sleep, faction-aware destinations, destination limiter. 8 arrival actions where NPCs consume real items from inventory (food, cigarettes, drinks, medkits) or trade artefacts for AP ammo, grenades, and supplies.
    Pipeline: split radiant/reactive event limiters, consequence token bucket, actor_on_interaction adapter for idle on-map squads. Reactive events (deaths, medkits, artefact pickups) no longer starved by radiant evaluation. Player-observed events increase significantly.
    Removed: global cause lock, ap_stats, xgoap_anim (vanilla gulag handles animations)
    Fixed: reactive event starvation
    Fixed: cause lock blocking 90% of predicate passes, starving needs and area causes
    Fixed: wounded_help faction matching (actor_stalker prefix stripping)
    Fixed: PDA messages now distinguish stalkers from mutants
    Fixed: supply_trader exchange giving free items (now requires artefact)
    MCM: full reset button in General tab (resets all AlifePlus settings across every tab and sub-tab)
    MCM: cleaned up labels and descriptions (units moved to tooltip, gate numbers corrected)
    Perf: observability (debug/trace/observe) function-swap - zero per-call overhead at WARN level

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
