# AlifePlus: Emergent A-Life for STALKER Anomaly

You are not special.

A simulation layer for A-Life, built on a reactive engine: engine callbacks become causes, causes dispatch consequences, consequences chain into new causes, with no spawning or teleporting.
On that runs an emergent economy. Squads trade, barter, loot corpses and stock stashes, and what a faction buys and sells reshapes what its traders put on the shelf for you.
Squads are tracked off-map, marked on the PDA and reported as news. Any "when X happens, do Y" scenario registers as a cause and a consequence. See the [integration guide](doc/integration.md).

[ModDB](https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01) | [Nexus](https://www.nexusmods.com/stalkeranomaly/mods/105) | [Releases](https://github.com/damiansirbu-stalker/AlifePlus/releases) | [Bugs, suggestions](https://github.com/damiansirbu-stalker/AlifePlus/issues)

Requires: Anomaly 1.5.3, modded exes (themrdemonized or AOEngine), [xlibs](https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001), MCM. Exact versions in [readme.txt](doc/readme.txt).

## Alife Collection

- [AlifeAmbience](https://github.com/damiansirbu-stalker/AlifeAmbience)
- [AlifeBalance](https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance)
- [AlifeCompanions](https://github.com/damiansirbu-stalker/AlifeCompanions)
- [AlifeDiegetic](https://www.moddb.com/mods/stalker-anomaly/addons/diegetic-audio-control-100)
- [AlifeGuard](https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001)
- [AlifePlus](https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01)
- [AlifeSpooks](https://github.com/damiansirbu-stalker/AlifeSpooks)
- [AlifeTactics](https://www.moddb.com/mods/stalker-anomaly/addons/alifetactics)
- [FurnitureFuel](https://github.com/damiansirbu-stalker/FurnitureFuel)
- [JitProfiler](https://github.com/damiansirbu-stalker/JitProfiler)
- [TestZone](https://github.com/damiansirbu-stalker/TestZone)
- [xlibs](https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)

## Documentation

- [readme.txt](doc/readme.txt) - full description, features
- [changelog](doc/changelog) - version history
- [manifesto.md](doc/manifesto.md) - design rationale, with GSC developer quotes and engine source evidence
- [architecture.md](doc/architecture.md) - event pipeline, dispatch pipeline, protection, ownership, lifecycle
- [integration.md](doc/integration.md) - how to build on AlifePlus: causes, consequences, API reference
- [conventions.md](doc/conventions.md) - naming rules, result codes, MCM settings, logging format

## Conflicting mods

AlifePlus rides vanilla NPC behavior, meaning looting corpses, gathering items, helping wounded and fighting.
Mods that disable those behaviors leave AlifePlus with less to react to.
Look for overlays of `gamedata/scripts/xr_*.script` or `gamedata/configs/ai_tweaks/*.ltx` that turn schemes off.

## License

PolyForm Perimeter License. Other mods can depend on, call, and ship with AlifePlus in modpacks, with visible credit.
What is not allowed: cloning the architecture, reverse-engineering internal systems, or reproducing the implementation in a competing mod.
See [LICENSE](LICENSE) and the [integration guide](doc/integration.md).

A report documenting unauthorized reproduction of this codebase is at https://damiansirbu-stalker.github.io/siski-report/
