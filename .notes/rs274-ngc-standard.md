---
id: rs274-ngc-standard
title: RS274/NGC G-code standard
type: reference
schema_version: 1
created: 2026-06-14T12:40:00Z
updated: 2026-06-14T12:40:00Z
valid_until: null
author: claude
session: null
tags: [cnc, gcode, standard, rs274, ngc]
aliases: [rs274, ngc, rs274/ngc, rs274-ngc, gcode standard, cnc gcode standard]
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
---

# RS274/NGC G-code standard

> RS274/NGC is the mandatory naming and behavior reference for all CNClipper module APIs — G/M-code semantics, parameter names, and capability definitions must trace back to this standard.

## Context

RS274/NGC is the NIST-published G-code standard that defines the core G/M-code vocabulary used across CNC controllers (mill, lathe, waterjet, plasma, laser, etc.). CNClipper's `.standards.md` mandates that all module APIs be grounded in RS274/NGC concepts — if a capability maps to an existing G/M-code concept, the module must use that naming and behavior, not a machine-specific one.

The standard is the primary authority; LinuxCNC's documentation is the secondary reference for CNC controller semantics where RS274/NGC leaves behavior underspecified.

## Observations

- [registry] Mandated in `.standards.md#Design-Principle-Machine-Agnostic` as the naming/behavior reference for all CNClipper module APIs #standard
- [registry] Core G-codes explicitly used or planned: G4 (dwell), G10 L2/L20 (WCS set), G53 (machine coords), G54-G59/G59.1-G59.3 (WCS select), G92/G92.1 (temp offset) #gcode
- [registry] Core M-codes explicitly used or planned: M3/M4/M5 (tool on/off), M62-M68 (I/O — M64/M65 digital, M66 input wait, M68 analog), M3/M5 with S parameter (spindle speed) #mcode
- [registry] Validation rule from standards: before finalizing any module API, describe ≥3 different machine types using that API — if the API can't express all three cleanly, the abstraction is wrong #rule
- [registry] Phase 1 modules trace directly to RS274/NGC: cnc_wcs.py (G54-G59, G10, G53, G92), cnc_io.py (M64-M68), cnc_tool_output.py (M3/M4/M5) #architecture

## Open Questions

- Which specific RS274/NGC version/document the standard references is not stated — NIST RS274/NGC v3 (2000) is the common base; LinuxCNC's G-code reference extends it.

## Relations

- relates-to [[cnclipper-source-202606120638]] (standard mandated for this repo's module APIs)
- relates-to [[linuxcnc-reference]] (secondary reference for CNC controller semantics)
