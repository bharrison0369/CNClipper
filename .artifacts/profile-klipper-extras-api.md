---
id: klipper-extras-api
title: Klipper klippy/extras/ module API
schema_version: 2
created: 2026-06-14T12:40:00Z
updated: 2026-06-14T12:40:00Z
valid_until: null
author: claude
session: null
tags: [klipper, python, api, extras, firmware, cnc]
aliases: [klipper extras, klippy extras, klipper module api, klipper plugin api, klippy/extras]
status: active
supersedes: null
confidence: 60
source_basis: document
human_edited: false
sensitivity: normal
decisions: []
model: claude-sonnet-4-6
model_basis: confirmed
provenance:
  harvest: deterministic
  recall-extract: claude-sonnet-4-6
  find-missing: claude-sonnet-4-6
  precision-judge: claude-sonnet-4-6
artifact_kind: memory
memory_class: semantic
semantic_kind: state
scope: CNClipper
---

# Klipper klippy/extras/ module API

> The `klippy/extras/` Python API is the integration surface for all CNClipper modules — every CNC capability is implemented as a standalone extras module using this API; the patterns are non-obvious and documented here for cross-session continuity.

## Context

Klipper's `klippy/extras/` system allows adding new G/M-code handlers and machine capabilities as standalone Python files without modifying Klipper core. CNClipper's entire implementation strategy is addon-first via this API. The Phase 1 implementation plan (`docs/plans/2026-04-11-phase1-core-primitives.md`) contains the most detailed documentation of these API patterns in the corpus — this note extracts the durable cross-session facts.

Modules are discovered by Klipper when referenced in `printer.cfg` as a config section matching the module filename.

## Observations

- [lore] Entry point: `load_config(config)` — called by Klipper when the config section is found; return an instance; all config parsing MUST happen in `__init__` (unread params cause errors at startup) #api
- [lore] `load_config_prefix(config)` — variant for numbered sections like `[cnc_io port0]`; use when a module supports multiple instances #api
- [lore] Config reading: `config.get()`, `config.getfloat()`, `config.getint()`, `config.getboolean()` — always call with `default=` for optional params; omitting default causes startup error if key absent #api
- [lore] G/M-code registration: `gcode.register_command(name, handler, desc=)` where handler takes a `gcmd` object; G/M-code names must be valid identifiers — dots not allowed, e.g. G59.1 cannot be registered directly; workaround: use a generic `CNC_WCS_SELECT INDEX=6` command #api
- [lore] Cross-module lookup: `printer.lookup_object('module_name')` — safe only in `klippy:connect` handler (modules may not be loaded at `__init__` time); use `printer.register_event_handler('klippy:connect', cb)` for deferred init #api
- [lore] Always-available objects: `printer.lookup_object('gcode')` and `printer.lookup_object('pins')` are safe in `__init__` #api
- [lore] Pin allocation: `ppins.setup_pin('digital_out', pin_name)` then `pin.setup_max_duration(0)` + `pin.setup_start_value(start, shutdown)` before first use; `setup_cycle_time(t)` required for PWM pins #api
- [lore] GCode command handler signature: `def cmd_NAME(self, gcmd)` where gcmd has `get_float()`, `get_int()`, `get()`, `respond_info()`, and `gcmd.error()` for raising errors #api
- [lore] WCS persistence pattern: use `printer.lookup_object('save_variables')` (Klipper built-in); access `allVariables` dict; write back via `gcode.run_script_from_command("SAVE_VARIABLE VARIABLE='key' VALUE='repr'")` #pattern
- [lore] SET_GCODE_OFFSET is the mechanism for applying WCS offsets: `gcode.run_script_from_command("SET_GCODE_OFFSET X=... Y=... Z=...")` — applied to all subsequent moves #pattern
- [lore] Mock harness for testing (Phase 1 plan): `MockPrinter`, `MockGcode`, `MockPins`, `MockMcuPin`, `MockConfig`, `MockGcmd` classes in `tests/conftest.py` replicate the Klipper API surface for pytest unit tests without running Klipper #testing

## Open Questions

- Motion-synchronized I/O (M62/M63) requires planner integration — how/whether Klipper exposes a public hook for this is unresolved (Phase 2 deferred item, high-risk feasibility).
- Feed hold implementation path — likely needs a core hook or event; not yet prototyped.
- Input pin reading (M66 L1-L4 wait modes) — requires Klipper's endstop or filament_switch API; not yet wired in Phase 1.

## Relations

- relates-to [[cnclipper-source-202606120638]] (this is the API the repo is built on)
- relates-to [[klipper-firmware]] (the firmware whose extras API this describes)
