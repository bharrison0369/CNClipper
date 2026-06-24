---
id: cnclipper-source-202606120638
title: CNClipper — source entity and repo manifest
schema_version: 2
created: 2026-06-12T06:38:00Z
updated: 2026-06-23T00:00:00Z
path: C:\GitHub\CNClipper
status: active
account: personal
reachable_via: local
tags: [cnc, klipper, python, hobby, open-source]
aliases: [cnc klipper, klipper cnc modules, cnclipper, cnc module library]
lifecycle: paused
artifact_kind: reference
model: unattributed
model_basis: unattributed
edited_by: claude-opus-4-8 (2026-06-23, decomposition-staleness remediation — repointed deleted-monolith body refs, backfilled Mentioned by)
author: claude
scope: CNClipper
---

# CNClipper — source entity and repo manifest

> Universal reusable CNC module library for Klipper (`klippy/extras/` Python), currently at Phase 0 (scope lock / feasibility) with no implementation code; personal hobby/community project, dormant since 2026-04-15.

## Context

Personal (`bharrison0369`) hobby project targeting open-source community adoption. Provides reusable `klippy/extras/` Python modules for CNC machine classes (waterjet, plasma, laser, mill, EDM, lathe, etc.). GPL-3.0, matching Klipper. Wazer is the first hardware proving target (user owns a Wazer waterjet; Wazer-specific repo exists separately).

Last substantive work: 2026-04-15 (CODEX: scope-lock documentation — `docs/phase-0-scope-lock.md`, plus the then-live `.standards.md` and `.project-context.md` monoliths, `README.md`). Those two monoliths have since been decomposed/superseded into the `.artifacts\` notes (`cnclipper-module-design-standards`, `cnclipper-safety-standards`, `cnclipper-phase0-state`); `docs/phase-0-scope-lock.md` survives as the standing doc. Phase 1 implementation plan exists (`docs/plans/2026-04-11-phase1-core-primitives.md`) but is unstarted. 2026-04-22 and 2026-05-09 changelog entries are bulk maintenance sweeps (identity/harness, not development).

Status call: **dormant** — pre-implementation, all work is documentation and planning; no code exists; no active development signal.

## Key architecture decisions

- **Addon-first**: `klippy/extras/` modules first; identify hooks if needed; narrow patch/fork only after failed feasibility. (source: [[cnclipper-module-design-standards]], `docs/phase-0-scope-lock.md`)
- **Machine-agnostic abstractions**: all APIs validated against ≥3 machine types before finalizing; naming from RS274/NGC, LinuxCNC, grblHAL. (source: [[cnclipper-module-design-standards]])
- **Repo boundary**: shared CNC capabilities only; Wazer-specific material in `Wazer` repo. (source: [[cnclipper-phase0-state]], `docs/phase-0-scope-lock.md`)
- **Provisional 4-tier module model**: core/ common/ machine/ machines/ — may be revised after Phase 0. (source: [[cnclipper-module-design-standards]])

## Phase 0 open work

- Capability matrix not yet built.
- Five high-risk feasibility spikes not yet attempted: feed hold, motion-synchronized I/O, general probing, WCS beyond Klipper offsets, M3/M4/M5 interception.
- Fork decision pending (driven by failed feasibility spikes, not architecture preference alone).
(source: [[cnclipper-phase0-state]] Open work / High-risk feasibility items, `docs/phase-0-scope-lock.md#High-Risk-Feasibility-Items`)

## Related

Sibling: `Wazer` repo (Wazer-specific `printer.cfg`, hardware mapping, validation — not yet ACC-onboarded).

## Mentioned by

- [[cnclipper-phase0-state]] (repo manifest / orientation)
- [[cnclipper-critical-rules]] (repo the rules govern)
- [[cnclipper-module-design-standards]] (repo these standards govern)
- [[cnclipper-safety-standards]] (repo this safety set governs)
- [[klipper-extras-api]] (the API this repo is built on)
- [[rs274-ngc-standard]] (standard mandated for this repo's module APIs)
- [[linuxcnc-reference]] (standard referenced for this repo's I/O module design)
- [[recovered-2026-04-11-d5e20ea7]] (Phase 1 planning session)
