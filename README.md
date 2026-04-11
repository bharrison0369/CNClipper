# CNClipper

Universal CNC module library for Klipper — waterjet, plasma, laser, mill, EDM, and more.

CNClipper extends Klipper with CNC capabilities through `klippy/extras/` modules. No fork required — install alongside stock Klipper, configure only the modules you need.

## Status

**Phase: Architecture Design** — defining module boundaries, APIs, and machine-type requirements before writing code.

## What This Is

A library of Klipper modules that adds CNC functionality:

- **Core CNC features** — work coordinate systems (G54-G59), feed hold, real-time overrides, probing cycles
- **Machine control** — spindle, coolant, tool management, output sequencing, safety interlocks
- **Machine-specific modules** — waterjet, plasma, laser (CO2/fiber/diode), mill/router, EDM, lathe, and more

Each module is a standard `klippy/extras/` Python file that only loads when referenced in `printer.cfg`. Zero impact on configurations that don't use it.

## Architecture

```
CNClipper/
  core/         Tier 1 — Every CNC needs these
  common/       Tier 2 — Most machines need, configure per type
  machine/      Tier 3 — Shared by 2-3 machine classes
  machines/     Tier 4 — Machine-specific modules
```

See [docs/architecture.md](docs/architecture.md) for the full module design.

## Installation

> Not yet available — architecture design phase. Check back or watch this repo.

```ini
# Future: add to moonraker.conf for auto-updates
[update_manager CNClipper]
type: git_repo
path: ~/CNClipper
origin: https://github.com/bharrison6/CNClipper.git
```

## Machine Types

| Machine | Module | Status |
|---------|--------|--------|
| Waterjet | `[waterjet]` | In Development |
| Plasma | `[plasma]` | Planned |
| CO2 / Fiber / Diode Laser | `[cnc_laser]` | Planned |
| Mill / Router | Uses core + spindle + coolant | Planned |
| Wire EDM | `[edm_wire]` | Planned |
| Sinker EDM | `[edm_sinker]` | Planned |
| Lathe | `[lathe]` | Planned |
| Drag Knife | `[cnc_tangential]` | Planned |
| Foam Cutter | Uses core + output_sequencer | Planned |
| Pick and Place | `[pnp]` | Planned |

## Contributing

This project is designed to be community-driven. Each machine type can be developed independently by someone with access to that hardware. See [docs/contributing.md](docs/contributing.md) and [docs/testing-status.md](docs/testing-status.md).

## License

GPL-3.0 — same as Klipper.
