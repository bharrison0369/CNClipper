---
id: cnclipper-source-202606120638
title: CNClipper — source entity and repo manifest
type: reference
schema_version: 1
created: 2026-06-12T06:38:00Z
updated: 2026-06-12T06:38:00Z
path: C:\GitHub\CNClipper
status: dormant
account: personal
reachable_via: local
tags: [cnc, klipper, python, hobby, open-source]
aliases: [cnc klipper, klipper cnc modules, cnclipper, cnc module library]
---

# CNClipper — source entity and repo manifest

> Universal reusable CNC module library for Klipper (`klippy/extras/` Python), currently at Phase 0 (scope lock / feasibility) with no implementation code; personal hobby/community project, dormant since 2026-04-15.

## Context

Personal (`bharrison0369`) hobby project targeting open-source community adoption. Provides reusable `klippy/extras/` Python modules for CNC machine classes (waterjet, plasma, laser, mill, EDM, lathe, etc.). GPL-3.0, matching Klipper. Wazer is the first hardware proving target (user owns a Wazer waterjet; Wazer-specific repo exists separately).

Last substantive work: 2026-04-15 (CODEX: scope-lock documentation — `docs/phase-0-scope-lock.md`, `.standards.md`, `.project-context.md`, `README.md`). Phase 1 implementation plan exists (`docs/plans/2026-04-11-phase1-core-primitives.md`) but is unstarted. 2026-04-22 and 2026-05-09 changelog entries are bulk maintenance sweeps (identity/harness, not development).

Status call: **dormant** — pre-implementation, all work is documentation and planning; no code exists; no active development signal.

## Key architecture decisions

- **Addon-first**: `klippy/extras/` modules first; identify hooks if needed; narrow patch/fork only after failed feasibility. (source: .standards.md, docs/phase-0-scope-lock.md)
- **Machine-agnostic abstractions**: all APIs validated against ≥3 machine types before finalizing; naming from RS274/NGC, LinuxCNC, grblHAL. (source: .standards.md)
- **Repo boundary**: shared CNC capabilities only; Wazer-specific material in `Wazer` repo. (source: .project-context.md, docs/phase-0-scope-lock.md)
- **Provisional 4-tier module model**: core/ common/ machine/ machines/ — may be revised after Phase 0. (source: .standards.md)

## Phase 0 open work

- Capability matrix not yet built.
- Five high-risk feasibility spikes not yet attempted: feed hold, motion-synchronized I/O, general probing, WCS beyond Klipper offsets, M3/M4/M5 interception.
- Fork decision pending (driven by failed feasibility spikes, not architecture preference alone).
(source: .project-context.md#TODO, docs/phase-0-scope-lock.md#High-Risk-Feasibility-Items)

## Related

Sibling: `Wazer` repo (Wazer-specific `printer.cfg`, hardware mapping, validation — not yet ACC-onboarded).

## Mentioned by

<!-- accumulates linked notes -->
