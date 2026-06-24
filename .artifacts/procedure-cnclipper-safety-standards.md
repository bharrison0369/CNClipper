---
id: cnclipper-safety-standards
artifact_kind: memory
memory_class: procedural
schema_version: 2
title: CNClipper safety standards (fail-safe, output sequencing, watchdog integration)
created: 2026-06-23T19:00:00Z
updated: 2026-06-23T19:00:00Z
author: claude
model: claude-opus-4-8
model_basis: confirmed
status: active
load_profile: scope_entry
scope: CNClipper
source_rel: CNClipper\.standards.md
enforceability: mandatory
tags: [cnclipper, cnc, safety, fail-safe, watchdog]
aliases: [cnclipper safety rules, cnc fail-safe rules, output sequencing safety]
entities: [cnclipper]
source_basis: document
---

# CNClipper safety standards (fail-safe, output sequencing, watchdog integration)

> The safety requirements for CNClipper modules that drive dangerous process outputs (waterjet, plasma, laser, spindle). Mandatory — these protect hardware and operators. Pulled out as its own artifact because it is the highest-consequence rule set in the repo.

## Context

The Safety / Warnings section of `.standards.md`, governing any module that controls a dangerous process output. Reinforces the protocol-level safety hard rule.

## Rules

- Safety-critical modules must fail safe when sensor or state information is missing or ambiguous.
- Output sequencing must complete or abort cleanly and must not leave dangerous outputs in partial states.
- Machine-specific safety layers must document what happens if a sensor fails or reports stale data.
- Never assume hardware state without confirmation.
- Dangerous process outputs should integrate with Klipper watchdog/disconnect behavior where possible.

## Relations

- relates-to [[cnclipper-source]] (repo this governs)
- relates-to [[cnclipper-critical-rules]] (where this appears as a protocol hard rule)
- relates-to [[cnclipper-module-design-standards]] (rest of the same standards file)
