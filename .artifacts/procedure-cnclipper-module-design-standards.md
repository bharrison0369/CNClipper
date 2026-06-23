---
id: cnclipper-module-design-standards
artifact_kind: memory
memory_class: procedural
schema_version: 2
title: CNClipper module design standards (tiers, design rules, Klipper API conventions, naming)
created: 2026-06-23T19:00:00Z
updated: 2026-06-23T19:00:00Z
author: claude
model: claude-opus-4-8
model_basis: confirmed
status: active
load_profile: scope_entry
scope: CNClipper
source_rel: CNClipper\.standards.md
enforceability: preferred
tags: [cnclipper, klipper, python, standards, architecture, naming]
aliases: [cnclipper module tiers, cnclipper coding standards, klipper api conventions, cnclipper naming conventions]
entities: [cnclipper]
source_basis: document
---

# CNClipper module design standards (tiers, design rules, Klipper API conventions, naming)

> How CNClipper modules are organized, written, and named — the provisional 4-tier model, the module design rules, Klipper API conventions, and naming conventions. Preferred guardrails (the user may override case-by-case; agents may propose changes but must not change standards without explicit approval).

## Context

The technical/architecture rules from `.standards.md`. Project overview: CNClipper is a universal CNC module library for Klipper, Python 3 host-side modules matching Klipper conventions, GPL-3.0, addon-first integration through `klippy/extras/`. No build step by default (modules load at runtime); test command is TBD (align with Klipper host-side testing); a future `install.sh` should link modules into `klippy/extras/`.

## Module tiers (provisional, Phase 0)

1. `core/` — universal CNC primitives.
2. `common/` — composed patterns reused across many machine classes.
3. `machine/` — features shared by a smaller subset of machine classes.
4. `machines/` — thin machine-specific integration.

This tier model is provisional during Phase 0 and may be revised if it proves more confusing than useful (open Phase 0 decision: whether the `machine/`/`machines/` split survives).

## Module design rules

- Each module should be a standalone `klippy/extras/` Python file unless a clear reason exists to organize shared helper code separately.
- Modules must only load when referenced in config.
- Modules must not modify Klipper core files unless the user has explicitly approved a core-patch path.
- Cross-module dependencies must be explicit, through printer-object lookups.
- All registered G-code/M-code commands must be documented in module docstrings and user-facing docs.
- Shared modules should own controller semantics, not machine recipes (e.g. a waterjet pierce sequence belongs in the machine repo if it composes from shared tool control, I/O, and dwell primitives).

## Klipper API conventions

- Follow Klipper coding style and object patterns where practical.
- Use `config.get*()` methods for config parsing during module construction.
- Register commands through the Klipper gcode object.
- Use event handlers for late binding and startup validation.
- Use Klipper pin abstractions instead of direct GPIO access.

## Naming conventions

- Module files: `cnc_<feature>.py` for shared modules, `<machine_type>.py` for thin machine integrations.
- Config sections: `[cnc_<feature>]` for shared modules, `[<machine_type>]` for machine-specific integrations.
- G-code commands: use standard G/M-codes where they already exist; prefix truly custom commands with a clear namespace.
- Python classes: PascalCase. Config keys: snake_case.

## Rule governance

Standards are default guardrails, not a ban on new ideas. Agents may propose changes when they identify a materially better architecture, safety model, or maintenance strategy, but must not change standards without explicit user approval; the user may override any individual rule case-by-case. When a standard changes: update `.standards.md`, update dependent repo-local docs, log a `[CONFIG]`/`[DOCS]` entry in `.changelog.md`, and note the reason.

## Relations

- relates-to [[cnclipper-source-202606120638]] (repo these standards govern)
- relates-to [[cnclipper-critical-rules]] (the hard-rule summary of these)
- relates-to [[rs274-ngc-standard]] (the CNC naming/behavior reference these defer to)
- relates-to [[cnclipper-safety-standards]] (safety section of the same standards file)
