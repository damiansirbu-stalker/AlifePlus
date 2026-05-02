# AlifePlus: Emergent A-Life for STALKER Anomaly

You are not special.

AlifePlus is a reactive alife framework for STALKER Anomaly - a complete simulation layer that replaces passive A-Life with event-driven emergent behavior. It intercepts engine events, classifies them into causes through world-state predicates, and dispatches consequences that change the simulation. The simulation runs independent of the player: engine callbacks fire causes, causes dispatch consequences, consequences chain into new causes. Without spawning, teleporting, or faking.

Any alife scenario that can be described as "when X happens, do Y" can be implemented by registering a cause and a consequence. See the [integration guide](doc/integration-guide.md) for examples, API reference, and a two-mod collaboration scenario.

[ModDB](https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus) | [Releases](https://github.com/damiansirbu-stalker/AlifePlus/releases) | [Bugs, suggestions](https://github.com/damiansirbu-stalker/AlifePlus/issues)

Requires: Anomaly 1.5.3, Modded exes, [xlibs 1.2.0](https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001), MCM

## Documentation

- [readme.txt](doc/readme.txt) - full description, features
- [changelog](https://github.com/damiansirbu-stalker/AlifePlus/blob/main/doc/changelog) - version history
- [manifesto.md](doc/manifesto.md) - design rationale with GSC developer quotes and engine source evidence
- [architecture.md](doc/architecture.md) - system internals: event pipeline, dispatch pipeline, protection, ownership, lifecycle
- [integration-guide.md](doc/integration-guide.md) - how to build on AlifePlus: register causes/consequences, mod collaboration, API reference
- [conventions.md](doc/conventions.md) - naming rules, result codes, MCM settings, logging format

## License

PolyForm Perimeter License. Other mods can depend on, call, and ship with AlifePlus in modpacks, with visible credit. What is not allowed: cloning the architecture, reverse-engineering internal systems, or reproducing the implementation in a competing mod. See [LICENSE](LICENSE) and [usage guide](https://damiansirbu-stalker.github.io/siski-report/guide.html).

A report documenting unauthorized reproduction of this codebase is available at https://damiansirbu-stalker.github.io/siski-report/
