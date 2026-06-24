---
id: linuxcnc-reference
title: LinuxCNC (CNC controller reference)
schema_version: 2
created: 2026-06-14T12:40:00Z
updated: 2026-06-14T12:40:00Z
valid_until: null
author: claude
session: null
tags: [cnc, linuxcnc, controller, reference, gcode]
aliases: [linuxcnc, linux cnc, emc2, grblhal, grbl hal]
status: active
supersedes: null
confidence: 55
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
lifecycle: active
artifact_kind: memory
memory_class: semantic
semantic_kind: entity_profile
scope: CNClipper
---

# LinuxCNC (CNC controller reference)

> LinuxCNC is the secondary reference standard for CNClipper — used for CNC controller semantics and I/O models where RS274/NGC is underspecified; grblHAL is also referenced as a third authority.

## Context

LinuxCNC is an open-source CNC controller (formerly EMC2) with a rich G/M-code implementation and well-documented I/O model. CNClipper's `.standards.md` cites LinuxCNC alongside RS274/NGC as an authoritative reference for "CNC controller semantics and I/O models." When designing module behavior, CNClipper should verify alignment with LinuxCNC's interpretation of the standard.

grblHAL (a modern GRBL fork with HAL plugin architecture) is also cited in `.standards.md` as a third reference, particularly useful for embedded/real-time CNC controller patterns. It is grouped with LinuxCNC in the "reference CNC standards for naming and behavior" requirement.

The M64/M65/M66/M68 I/O M-codes used in `cnc_io.py` come directly from LinuxCNC's M-code vocabulary.

## Observations

- [registry] Referenced in `.standards.md` alongside RS274/NGC as design reference for "CNC controller semantics and I/O models" #standard
- [registry] M64 (digital out ON), M65 (digital out OFF), M66 (wait on digital input), M68 (set analog output) — these M-codes are LinuxCNC-specific extensions not in base RS274/NGC #mcode
- [registry] grblHAL also cited in same standards clause as third reference authority; included as alias here since it plays a secondary role to LinuxCNC in CNClipper docs #standard
- [registry] LinuxCNC's HAL (Hardware Abstraction Layer) I/O model is the semantic template for CNClipper's `cnc_io.py` port-numbered I/O design #architecture
- [registry] Phase 2 M-codes (M62/M63 — motion-synchronized I/O) are LinuxCNC-originated; deferred pending Klipper planner integration #deferred

## Open Questions

- Whether CNClipper will align with LinuxCNC's specific parameter conventions for all M-codes (e.g., M66 L-mode values) or adapt them for Klipper constraints — Phase 1 `cnc_io.py` uses L0 only for now.

## Relations

- relates-to [[cnclipper-source-202606120638]] (standard referenced for this repo's I/O module design)
- relates-to [[rs274-ngc-standard]] (primary standard; LinuxCNC extends it)
