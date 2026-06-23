---
id: cnclipper-critical-rules
artifact_kind: memory
memory_class: procedural
schema_version: 2
title: CNClipper critical rules (addon-first, machine-agnostic, repo boundary, safety, no-UI)
created: 2026-06-23T19:00:00Z
updated: 2026-06-23T19:00:00Z
author: claude
model: claude-opus-4-8
model_basis: confirmed
status: active
load_profile: scope_entry
scope: CNClipper
source_rel: CNClipper\.protocol.md
enforceability: mandatory
tags: [cnclipper, cnc, klipper, rules, protocol]
aliases: [cnclipper hard rules, addon-first rule, machine-agnostic rule, cnclipper repo boundary]
entities: [cnclipper]
source_basis: document
---

# CNClipper critical rules (addon-first, machine-agnostic, repo boundary, safety, no-UI)

> The five hard rules every agent must honor in CNClipper — they gate how modules are integrated, abstracted, scoped, and made safe. Mandatory: do not violate without explicit user approval.

## Context

These rules are the `@always` critical block from `.protocol.md`, each tracing to a design principle in `.standards.md` or the Phase 0 scope-lock doc. They are guardrails for any implementation or design work in this repo.

## Rules

- **Addon-first (hard rule).** Implement every feature as a `klippy/extras/` Python module first. Never touch Klipper core files unless the user has explicitly approved a core-patch path. Default order: extras module → identify missing hooks if the addon layer can't express it cleanly → narrow Klipper patch or fork only as last resort, kept as small as possible. A broad fork that redesigns Klipper into a machine-agnostic platform is out of scope for current work unless the user explicitly approves after feasibility work. (`.standards.md` Design Principle: Addon-First)
- **Machine-agnostic abstraction.** All module APIs must be describable against ≥3 machine types without machine-specific flags. Validate before finalizing any API: if it can't express three machine types cleanly without special cases, the abstraction is wrong. Ground naming/behavior in RS274/NGC, LinuxCNC, and grblHAL. (`.standards.md` Design Principle: Machine-Agnostic)
- **Repo boundary.** CNClipper owns reusable, shared CNC abstractions only. Wazer-specific `printer.cfg`, hardware mapping, validation, and process workflows stay in the `Wazer` repo — do not absorb them here even though Wazer is the first proving target. (`docs/phase-0-scope-lock.md`)
- **Safety (fail safe).** Safety-critical modules must fail safe on missing or ambiguous sensor/state information; output sequencing must complete or abort cleanly and never leave dangerous outputs in partial states; never assume hardware state without confirmation. (`.standards.md` Safety)
- **No UI work in this repo.** Browser-side and operator-interface work is a future, separate project. (`.standards.md`, `docs/phase-0-scope-lock.md`)

## Relations

- relates-to [[cnclipper-source-202606120638]] (repo this governs)
- relates-to [[rs274-ngc-standard]] (naming/behavior reference for the machine-agnostic rule)
- relates-to [[cnclipper-module-design-standards]] (the design rules these summarize)
- relates-to [[cnclipper-safety-standards]] (full safety section)
