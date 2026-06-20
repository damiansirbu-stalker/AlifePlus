# AlifePlus: Emergent A-Life for STALKER Anomaly

You are not special.

AlifePlus is a reactive alife framework for STALKER Anomaly - a complete simulation layer that replaces passive A-Life with event-driven emergent behavior. It intercepts engine events, classifies them into causes through world-state predicates, and dispatches consequences that change the simulation. The simulation runs independent of the player: engine callbacks fire causes, causes dispatch consequences, consequences chain into new causes. Without spawning, teleporting, or faking.

Any alife scenario that can be described as "when X happens, do Y" can be implemented by registering a cause and a consequence. See the [integration guide](doc/integration.md) for examples, API reference, and a two-mod collaboration scenario.

[ModDB](https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01) | [Nexus](https://www.nexusmods.com/stalkeranomaly/mods/105) | [Bugs, suggestions](https://github.com/damiansirbu-stalker/AlifePlus/issues)

Requires: Anomaly 1.5.3, [themrdemonized 2025.9.10+](https://github.com/themrdemonized/xray-monolith) or [AOEngine v0.55+](https://github.com/Mirrowel/AOEngine-Assets), [xlibs 1.7.7](https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001), MCM

Alife Collection:
- [AlifePlus](https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01)
- [AlifeBalance](https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance)
- [AlifeGuard](https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001)
- [AlifeTactics](https://www.moddb.com/mods/stalker-anomaly/addons/alifetactics)

## Documentation

- [readme.txt](doc/readme.txt) - full description, features
- [changelog](https://github.com/damiansirbu-stalker/AlifePlus/blob/main/doc/changelog) - version history
- [manifesto.md](doc/manifesto.md) - design rationale with GSC developer quotes and engine source evidence
- [architecture.md](doc/architecture.md) - system internals: event pipeline, dispatch pipeline, protection, ownership, lifecycle
- [integration.md](doc/integration.md) - how to build on AlifePlus: register causes/consequences, mod collaboration, API reference
- [conventions.md](doc/conventions.md) - naming rules, result codes, MCM settings, logging format

## Conflicting mods

AlifePlus rides vanilla NPC behavior — looting corpses, gathering items, helping wounded, fighting. Mods that disable those behaviors leave AlifePlus with less to react to. Look for overlays of `gamedata/scripts/xr_*.script` or `gamedata/configs/ai_tweaks/*.ltx` that turn schemes off.

## License

PolyForm Perimeter License. Other mods can depend on, call, and ship with AlifePlus in modpacks, with visible credit. What is not allowed: cloning the architecture, reverse-engineering internal systems, or reproducing the implementation in a competing mod. See [LICENSE](LICENSE) and [integration guide](doc/integration.md).

A report documenting unauthorized reproduction of this codebase is available at https://damiansirbu-stalker.github.io/siski-report/
