# CNClipper - Technical Standards

This file is the single source of truth for all technical rules in this repository.

**All agents (Claude, Gemini, Codex) MUST follow these standards.**

---

## Project Overview

- **Project:** CNClipper — Universal CNC module library for Klipper
- **Platform:** Klipper (klippy/extras/ modules)
- **Language:** Python 3 (matching Klipper host)
- **License:** GPL-3.0 (matching Klipper)

---

## Build & Test

- **No build step** — Python modules loaded by Klipper at runtime
- **Test command:** TBD — Klipper has a test framework in `klippy/test/`
- **Install:** `install.sh` symlinks modules into klippy/extras/

---

## Architecture

### Module Tiers

1. **core/** — Universal CNC features every machine type needs (WCS, feed hold, overrides, probing)
2. **common/** — Features most machines need, configured per type (spindle, coolant, tool table, interlocks)
3. **machine/** — Features shared by 2-3 machine classes (laser PWM, pierce cycle, THC, gap servo)
4. **machines/** — Machine-specific modules (waterjet, plasma, mill, EDM, lathe, etc.)

### Module Design Rules

- Each module is a standalone klippy/extras/ Python file
- Modules only load when referenced in printer.cfg (zero bloat for unused features)
- Modules MUST NOT modify Klipper core files — use only public klippy APIs
- Cross-module dependencies must be explicit (use `printer.lookup_object()`)
- All G-code/M-code registrations must be documented in module docstrings

### Klipper API Conventions

- Follow Klipper's existing coding style (see `klippy/extras/` examples)
- Use `config.get()`, `config.getfloat()`, `config.getint()` for config parsing
- Register G-code commands via `self.gcode.register_command()`
- Use `self.printer.register_event_handler()` for event hooks
- Pin access via `ppins.setup_pin()` — never direct GPIO

---

## Naming Conventions

- **Module files:** `cnc_<feature>.py` for core/common, `<machine_type>.py` for machine-specific
- **Config sections:** `[cnc_<feature>]` for core/common, `[<machine_type>]` for machine-specific
- **G-code commands:** Use standard G/M codes where they exist (G54, M3, etc.), prefix custom commands with machine type
- **Python classes:** PascalCase, matching Klipper convention (e.g., `CncWorkCoordinates`, `WaterjetController`)
- **Config keys:** snake_case, matching Klipper convention (e.g., `pierce_delay`, `door_switch_pin`)

---

## Safety / Warnings

- **Interlock modules are safety-critical** — they must fail-safe (deny motion/output if sensor state is unknown)
- **Output sequencing must be deterministic** — timed sequences must complete or abort cleanly, never leave outputs in partial state
- **Machine-specific safety modules must document all interlocks** with explicit "what happens if this sensor fails" analysis
- **Never assume hardware state** — always read/confirm before acting (e.g., confirm door closed, not just "door not reported open")
- **Watchdog integration** — modules that control dangerous outputs (laser, plasma, waterjet HP pump) must register with Klipper's watchdog so outputs disable on host disconnect

---

## Legacy Warnings

(None — new project)

---

## When Standards Change

1. Update this file (source of truth)
2. Test changes (build/test)
3. Log in `.agent-log\changelog.md` with `[CONFIG]` action type
4. Note what changed so other agents re-read configs
