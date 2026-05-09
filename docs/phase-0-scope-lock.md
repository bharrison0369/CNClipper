# CNClipper Phase 0 - Scope Lock and Feasibility

Last updated: 2026-04-15

## Purpose

Phase 0 exists to lock scope before implementation and to identify which CNC capabilities can be built cleanly as Klipper addons versus which may require core hooks or a fork.

This phase is documentation and feasibility work first. It is not a commitment to implement every CNC feature or every machine class.

## Project Intent

CNClipper is a reusable CNC capability layer for Klipper.

The goal is:

- stock Klipper
- plus CNClipper
- plus a machine-specific repo

That combination should provide what is needed to run a given machine.

CNClipper exists to prevent redundant code across machine-specific repos by capturing the capabilities that are common across multiple CNC machine classes.

## Repo Boundaries

CNClipper owns:

- reusable CNC abstractions
- Klipper integration for those abstractions
- shared controller semantics
- feature classification and feasibility work

CNClipper does not own:

- Wazer-specific `printer.cfg`
- Wazer hardware mapping and validation
- machine-specific process recipes
- UI work

Wazer-specific material stays in the `Wazer` repo, even when Wazer is the first proving target.

## Current Strategy

The implementation strategy is addon-first.

Default order of attack:

1. Try `klippy/extras/` modules.
2. If that is insufficient, identify the missing Klipper hooks.
3. If hooks are still not enough, consider a narrow core patch or fork.

The current plan is not to redesign Klipper into a machine-agnostic controller platform before proving the CNC requirements. A broader fork remains a possible future direction, but it is not the present execution plan.

## Validation Sequence

Validation should proceed in this order:

1. Existing working Klipper 3D printers for safe motion and API experiments.
2. Wazer as the first real CNC proving target.
3. Additional machine classes later if the abstractions prove reusable.

Wazer is the first proving target, not the architectural center of gravity.

## Scope Definition

The scope of CNClipper is not "all CNC machines" as direct deliverables.

The scope is the set of reusable capabilities needed across many CNC machine classes, including but not limited to:

- waterjet
- laser cutter / engraver
- plasma cutter
- drag knife
- oxy-fuel
- nibbler
- mill / router
- lathe
- wire EDM
- sinker EDM
- hybrid additive / subtractive
- 6-axis robot
- welding
- pick and place
- CMM
- wire bending

Many machine-specific workflows will remain outside CNClipper. The question for each candidate feature is whether it is common enough to belong in the shared layer.

## Inclusion Rule

A feature belongs in CNClipper when all of the following are true:

- stock Klipper does not cover it fully, or covers it in a printer-shaped way that is insufficient for CNC
- it is reusable across multiple machine classes
- it represents a controller capability, not a single-machine recipe

A feature should stay in a machine-specific repo when it is mostly a process sequence or workflow built from existing primitives.

Examples:

- `G4` dwell belongs in the shared layer as a universal primitive
- a waterjet pierce sequence usually does not need its own shared module if it can be expressed as shared tool control, I/O, and dwell
- plasma THC is likely too process-specific to be part of the first shared baseline

## Capability Classification Method

Each candidate capability should be classified across four dimensions:

1. Stock Klipper coverage
   - fully covered
   - partially covered
   - not covered
2. Reuse across machine classes
   - broad
   - moderate
   - narrow
3. Expected implementation path
   - macro-safe
   - extras-module
   - likely needs core hook
   - likely needs MCU work
4. Priority
   - baseline shared scope
   - later shared scope
   - machine-specific scope

## Initial Baseline Candidates

These are strong candidates for the baseline shared scope:

- work coordinate systems and offsets
- probing primitives
- synchronized and immediate I/O
- input wait semantics
- universal tool output semantics
- coolant / process output control
- feed hold
- feed and tool overrides
- tool table and tool length semantics
- path control modes
- canned-cycle framework if it proves broadly reusable

These are likely later or machine-specific unless Phase 0 proves otherwise:

- plasma THC
- laser PPI or raster-specific behavior
- EDM gap control
- welding process control
- robot-specific kinematics and planning

## High-Risk Feasibility Items

These are the first features that should be explored because they are the most likely to force core changes:

- true feed hold
- motion-synchronized I/O
- general probing semantics
- work coordinate systems beyond Klipper's current offset model
- universal `M3/M4/M5` interception and replacement

Fork risk should not be guessed from architecture discussion alone. It should be driven by attempts to implement the behavior cleanly.

## Fork Discussion

A machine-agnostic Klipper-derived platform may be valuable long-term. The host-plus-MCU split, timer synchronization, and multi-MCU architecture are all worth preserving.

However, the current recommendation is:

- do not rebuild Klipper first
- do not move 3D printing out into an addon before Phase 0 proves that direction is necessary
- do not create a broad fork until the addon-first path has been tested against the hard CNC requirements

If a fork becomes necessary, prefer a narrow hook-oriented fork first, not a full platform rewrite.

## Out of Scope

The following are out of scope for CNClipper at this stage:

- browser UI work
- a generic CAM pipeline
- machine-specific configs in this repo
- a full machine-agnostic Klipper rewrite before feasibility work

## Next Outputs

Phase 0 should produce:

- a feature matrix comparing stock Klipper coverage to CNC needs
- a prioritized list of baseline shared capabilities
- a risk classification for each major capability
- a decision on which early spikes should be attempted first
