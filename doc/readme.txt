AlifePlus: Emergent A-Life for STALKER Anomaly, by Damian
Latest: 1.1.0 (xlibs 1.0.4)

You are not special.

AlifePlus is a behavior mod that follows Roadside Picnic and the original STALKER, not "warfare" or "action" alternatives.

GSC described their vision: "We wanted a simulation where the game world operates independently of player actions."
I respected this and went further. In AlifePlus the simulation runs independent of the player, fair and equal for all creatures on the map, where the player is just one entity.

Why use AlifePlus? What makes it special?
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
- If the player kills it, he gets rewarded. If an NPC kills it, the NPC may become a legend... and the cycle continues

AlifePlus was built as a framework from the X-Ray engine source code and modding references.
Every callback the system uses was extracted from the engine's C++ source, documented with full API signatures, and measured in real time.
Where the engine lacks a callback, a runtime hook emits a synthetic event without modifying base scripts.
All engine work is extracted into xlibs, a shared library focused on doing things right, according to original source code.

Details of the system can be found inside architecture.md, but in a nutshell:
- purely reactive architecture
- nothing is scripted, nothing is iterated, forced or hijacked
- nothing is spawned out of thin air
- original xray events are intercepted, evaluated and aggregated into "causes"
- causes can be reactive, a direct reaction to an event
- causes can also be radiant, where the NPC evaluates its surroundings and acts on them
- causes are decoupled from, and lead to consequences - actual behavior
- because of the design, events, causes and consequences interact in a nondeterministic manner, leading to emergent gameplay

Performance impact is none. The attached screenshot shows zero performance hit with 1000x load, measured for almost 8 hours continuously. That being 2 months old code, latest is even more efficient.
Since I rely on the engine's heavy lifting and flow from the existing system, the performance hit is almost zero and everything is cohesive.

Nothing is faked and every reaction is earned from world state. Every creature, item or object involved was already there, living its own A-Life, and AlifePlus just extends what it is already doing.
The system works with original xray events, moves real entities across real distances, and unfolds consequences without forcing, spawning, or teleporting

Each major release contains a set of behaviors following a clear direction and game design.
For the first release I focused on increasing the risk/reward for all entities in the world and creating a heightened sense of danger.

Every cause and consequence is individually toggleable through MCM.
Chances, cooldowns, thresholds, and rate limits are all configurable.
Log level goes from silent to full tracing and pathing, performance timing, and PDA map markers on every operation.

Features:

Causes:
  Massacre - Reactive. Deaths pile up at a smart terrain.
  SquadKill - Reactive. A squad has been wiped out.
  BaseKill - Reactive. Deaths pile up at a faction base.
  Elite - Reactive. NPC accumulated enough kills to level up. 10 levels, each requiring more kills.
  EliteKill - Reactive. An elite has died.
  Wounded - Reactive. An NPC or player used a healing item.
  Harvest - Reactive. An NPC or player picked up an artefact.
  Stash - Radiant. Squad discovers nearby stashes.
  Area - Radiant. Squad discovers nearby unguarded area.

Consequences:
  Massacre: Scavenge - Nearby scavengers and predators converge on the massacre site.
  Massacre: Investigate - Victims' faction sends nearby squads to investigate.
  SquadKill: Revenge - Same-faction squads pursue the killer as it moves.
  SquadKill: Flee - Nearby squads of the victim's faction flee to the nearest base.
  BaseKill: Support - Nearby friendly squads rush to reinforce the base under attack.
  BaseKill: Flee - Squads at the base evacuate to the nearest faction base.
  Elite: Promote - Registers the NPC as elite. Rank boost, grenades, and AP ammo scale with level.
  EliteKill: Bounty - Killer receives rubles scaled by the elite's level.
  EliteKill: Targeted - Other elites on the same level pursue the killer as it moves.
  Stash: Loot - Discovering stalker squad moves to loot the stash.
  Stash: Ambush - Discovering outlaw squad camps at the stash. The stash is bait.
  Stash: Fill - Discovering stalker squad moves to hide supplies in the stash.
  Area: Conquer - Squad claims the empty smart terrain. Engine spawns their faction there.
  Wounded: Hunt - Nearby predator mutants move toward the wounded NPC or player.
  Wounded: Help - Nearby same-faction squads rush to help the wounded.
  Harvest: Hunt - Nearby outlaws pursue whoever picked up the artefact as they move.

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

Known issues:
  Map markers are a debugging feature (log_level=DEBUG). They lack persistence
  and do not update on entity death, despawn, or level transition.

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

Credits:
Stalker_Boss - Russian translation

Development:
Written against X-Ray Monolith engine source, Demonized exes source code,
and Anomaly 1.5.3 unpacked gamedata. Code patterns and engine usage validated against established work
by reputable GAMMA modders (Demonized, Vintar0, RavenAscendant, xcvb). The
code is validated in real time by a multi-stage pipeline: luacheck, selene,
tree-sitter AST analysis, contract rules, cross-file dependency
resolution, cyclomatic complexity analysis, crash and vulnerability pattern
detection, lua54 integration testing with X-Ray engine stubs, gitleaks
secret scanning. Full report in doc/test-report.log.

Versions:

  1.1.0
    10-level elite system replaces binary elite/legend. Going Speed balances simulation across maps.
    Added: Going Speed - X-Ray's two-rate A-Life split applied to incoming events. Bresenham ratio gate + class-isolated cooldowns. MCM slider (-10 to +10, default 8).
    Changed: elite system reworked from binary elite/legend to 10 levels (community requested)
    Changed: naming conventions standardized across causes, consequences, and MCM
    Changed: MCM General tab reorganized (distributor/cause/consequence sections, clearer labels)
    Changed: test tools moved from MCM buttons to console (ap_test)
    Added: AP ammo restock for elites, best ammo for their weapon (cooldown-gated)
    Added: engine rank boost by elite level (up to 20% accuracy at max rank)
    Added: result codes standardization (OK_STOP, OK_NEXT, RULES_NEXT, etc.)
    Removed: damage dealt/taken multipliers for elites (community requested)
    Changed: AP ammo restock uses box_size for correct round count per box
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
