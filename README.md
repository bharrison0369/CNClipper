# CNClipper

Universal CNC module library for Klipper - waterjet, plasma, laser, mill, EDM, lathe, and more.

CNClipper extends Klipper with CNC capabilities through `klippy/extras/` modules. The project is addon-first: prefer stock Klipper plus reusable CNC modules, and only consider core patches or a fork if Phase 0 proves they are necessary.

## Status

**Phase: Phase 0 scope lock and feasibility** - defining reusable CNC capabilities, repo boundaries, and implementation risk before writing code.

## What This Is

A library of Klipper modules that adds reusable CNC functionality:

- **Core CNC features** - work coordinate systems, probing, feed hold, coordinated I/O, overrides
- **Machine control** - tool output, coolant / process outputs, tool management, sequencing, safety interlocks
- **Shared abstractions** - capabilities that should not be reimplemented in every machine-specific repo

Each module is a standard `klippy/extras/` Python file that only loads when referenced in `printer.cfg`. Configurations that do not use CNC features should not pay for them.

## Documentation

- [Phase 0 scope lock](docs/phase-0-scope-lock.md)
- [Initial Phase 1 implementation plan](docs/plans/2026-04-11-phase1-core-primitives.md)

## Repo Boundaries

- `CNClipper` owns reusable CNC capabilities shared across machine classes.
- Machine-specific `printer.cfg`, hardware mapping, and validation belong in machine repos.
- Wazer-specific material stays in the `Wazer` repo.
- UI work is out of scope here.

## Machine Classes

| Machine Class | Status |
|---|---|
| Waterjet | First proving target |
| Plasma | Planned |
| CO2 / Fiber / Diode Laser | Planned |
| Mill / Router | Planned |
| Wire EDM | Planned |
| Sinker EDM | Planned |
| Lathe | Planned |
| Drag Knife | Planned |
| Foam Cutter | Planned |
| Pick and Place | Planned |

Additional target classes under consideration include oxy-fuel, nibbler, hybrid additive / subtractive systems, 6-axis robots, welding, CMM, and wire bending. CNClipper does not need to implement every machine class directly; it needs to capture the shared controller capabilities that those machines reuse.

## Validation

Early validation is expected to start on existing working Klipper 3D printers for safe motion and API testing, followed by Wazer as the first real CNC proving target.

## Contributing

This project is intended to be community-driven. The immediate work is Phase 0 scope lock and feasibility classification, followed by shared capability implementation once the boundaries are clear.

## License

GPL-3.0 - same as Klipper.
