---
id: cnclipper-phase0-state
artifact_kind: memory
memory_class: semantic
semantic_kind: state
schema_version: 2
title: CNClipper Phase 0 state — scope, open work, capability candidates, machine classes
created: 2026-06-23T19:00:00Z
updated: 2026-06-23T19:00:00Z
author: claude
model: claude-opus-4-8
model_basis: confirmed
status: active
lifecycle: paused
load_profile: scope_entry
scope: CNClipper
source_rel: CNClipper\.project-context.md
tags: [cnclipper, cnc, phase-0, capability-matrix, roadmap]
aliases: [cnclipper phase 0, cnclipper open work, cnclipper capability candidates, cnclipper machine classes]
entities: [cnclipper]
source_basis: document
---

# CNClipper Phase 0 state — scope, open work, capability candidates, machine classes

> Where CNClipper stands and what's open: Phase 0 (scope lock / feasibility), no implementation code yet, with the capability-classification work, high-risk feasibility spikes, pending decisions, and the machine-class design space all still to be done. Dormant since 2026-04-15.

## Status

Phase 0 — scope lock and feasibility. Active work was documenting repo boundaries, classifying shared CNC capabilities, and identifying which features implement cleanly as addons vs. which may need Klipper hooks or a fork. Pre-implementation; no code exists. Dormant since 2026-04-15 (scope-lock documentation); the 2026-04-22 and 2026-05-09 changelog entries are bulk maintenance sweeps, not development.

## Open work (TODO)

- Build the Phase 0 capability matrix: feature × stock-Klipper coverage × machine-class reuse × likely implementation path × priority.
- Classify baseline shared capabilities vs. machine-specific workflows.
- Classify each feature as macro-safe, extras-module candidate, likely-needs-core-hooks, or likely-needs-MCU-work.
- Prototype feasibility for the five high-risk items (below).
- Decide whether the 4-tier directory model survives Phase 0 classification.
- Full documentation cleanup pass after Phase 0 decisions settle (including stale references in older planning docs).

## High-risk feasibility items (prototype first to bound fork risk)

These are most likely to force core patches:
1. True feed hold.
2. Motion-synchronized I/O.
3. General probing semantics.
4. WCS beyond Klipper's current offset model.
5. Universal M3/M4/M5 interception and replacement.

## Baseline shared-capability candidates

Strong candidates for shared scope: WCS/offsets, probing primitives, synchronized and immediate I/O, input-wait semantics, universal tool output, coolant/process output, feed hold, feed/tool overrides, tool table/tool length, path-control modes.

## Pending decisions

- Whether a narrow Klipper hook patchset is sufficient for the hard CNC features.
- Whether any future fork stays hook-oriented or becomes a broader machine-agnostic platform effort.
- Whether the `machine/`/`machines/` split is worth keeping after capability classification.
- Which capabilities belong in baseline shared scope vs. later shared scope.

## Machine classes in view

Long-term design space (CNClipper need not implement each directly — it captures the shared capabilities across them): waterjet, laser cutter/engraver, plasma cutter, drag knife, oxy-fuel, nibbler, mill/router, lathe, wire EDM, sinker EDM, hybrid additive/subtractive, 6-axis robot, welding, pick-and-place, CMM, wire bending.

## Why CNClipper exists

The goal: stock Klipper + CNClipper + a machine-specific repo together give a complete path to running a machine without duplicating shared controller logic in every machine repo. CNClipper therefore focuses on reusable controller semantics, not machine recipes.

## Key docs

- Phase 0 scope lock: `docs/phase-0-scope-lock.md`.
- Phase 1 implementation plan (unstarted): `docs/plans/2026-04-11-phase1-core-primitives.md`.

## Relations

- relates-to [[cnclipper-source]] (repo manifest / orientation)
- relates-to [[cnclipper-critical-rules]] (rules constraining this work)
- relates-to [[rs274-ngc-standard]] (standard the Phase 1 modules trace to)
