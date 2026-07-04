---
id: machine-agnostic-design-phase1-plan
title: "Recovered: CNClipper machine-agnostic design principle discussion + Phase 1 plan seeding"
model: claude-sonnet-4-6
model_basis: confirmed
original_session_model: unattributed
schema_version: 2
created: 2026-06-21T00:00:00Z
updated: 2026-06-21T00:00:00Z
session: recovered-d5e20ea7
original_session_date: 2026-04-11
tags: [recovered, reconstructed, cnclipper, klipper, cnc, architecture, design]
aliases: [recovered-2026-04-11-d5e20ea7]
related: [cnclipper-source]
status: active
supersedes: null
confidence: 45
source_basis: transcript
human_edited: false
sensitivity: normal
decisions: []
artifact_kind: memory
memory_class: episodic
semantic_kind: state
scope: CNClipper
load_profile: on_demand
---

> **PROVENANCE GUARD** — This is a recovered digest, NOT a verbatim transcript. Assistant responses are permanently deleted. The 7 user prompts are ground truth. Changelog entries at 14:30 and 15:30 are indirect artifact evidence — they cite "Key design discussion with user" that corresponds to this session's 05:44–06:25 window, but those entries were written during a later session (a02ca962).
> A claim in a faithful section that lacks an artifact citation is NOT ground truth — treat it as prompt-derived (user intent) or narrative, never as a confirmed outcome.
> Reconstructing model: claude-sonnet-4-6 (confirmed); original session model: unattributed. See recovered-transcripts/CALIBRATION.md.

## From the user's prompts (ground truth — intent + user-stated facts)

- Session 2026-04-11 05:44 → 2026-04-13 09:14; 7 prompts. Real work: 05:44–06:25 (prompts [1]–[6]); prompt [7] is a `/resume` on 2026-04-13 at 09:14 (likely a re-open artifact).
- Asked agent to review the current development plan in the new CNClipper repo.
- Core design constraint stated by user (verbatim): "The main thing is this needs to be universal to pick up adoption and development by anyone other than me for my modded wazer (or maybe a half dozen other people who might want to mod their wazer). While I agree we don't need to make the half dozen other modules or even initialize them we need to keep the modules universal. We don't want to create a pierce module that works for waterjet but isn't viable for laser and is really just better described as a dwell from normal cnc code (not sure if thats accurate)."
- Asked to "make note the nuance that we just discussed for other agents to look at then do some research and create a refinement to the plan that currently exists."
- Confirmed assumption: "the what we discussed about keeping it machine agnostic is something that goes in standards.md?"
- Approved plan update; then: "Just save the plan for now. Lets wait for implementation."

## Artifact-cited outcomes (COMMIT / CHANGELOG / NOTE / PLAN)

**Indirect evidence — written by later session a02ca962, explicitly citing this session's discussion:**

CNClipper changelog `[2026-04-11 14:30]` CLAUDE [DESIGN]:
> "Added machine-agnostic design principle to STANDARDS.md — all modules must be grounded in universal CNC concepts, not machine-specific implementations. Includes validation rule (3-machine-type test) and CNC standard references. Key design discussion with user — abstractions must be universal to attract community adoption beyond Wazer. A 'pierce module' that only works for waterjet is wrong; it should be timed output sequencing that any machine configures. The 3-machine validation rule in STANDARDS.md enforces this."

CNClipper changelog `[2026-04-11 15:30]` CLAUDE [DESIGN]:
> "Wrote Phase 1 implementation plan: `docs/plans/2026-04-11-phase1-core-primitives.md`. 12 tasks with TDD steps covering: test infrastructure, G4 dwell evaluation, cnc_wcs, cnc_io, cnc_tool_output, install script, example configs for waterjet vs spindle."

Plan file `C:\GitHub\CNClipper\docs\plans\2026-04-11-phase1-core-primitives.md` — exists; created at 15:30 by the later session.

Curated note xref:
- `[[cnclipper-source]]` — machine-agnostic design principle, 3-machine-type validation rule, Phase 1 plan reference

**Important attribution note:** File modifications (STANDARDS.md update, plan creation) were executed by session a02ca962 (09:48–17:55 on 2026-04-11). This session generated the design discussion that seeded them; it produced no committed artifact of its own within its 05:44–06:25 active window.

## Inferred (low-confidence — do not distill as fact)

- Agent identified that the existing CNClipper plan used waterjet-specific language (e.g., "pierce module") inconsistent with the intended universal architecture. [basis: CHANGELOG 14:30 note + NEXT-PROMPT]
- Agent confirmed the pierce-as-G4-dwell framing: in RS274/NGC and grblHAL, pierce = timed dwell before cutting, universally meaningful across waterjet/laser/plasma/router. [basis: CHANGELOG 14:30 reference to CNC standards; DOMAIN]
- Agent researched RS274/NGC, grblHAL, LinuxCNC for module naming grounding. [basis: CHANGELOG 14:30 "CNC standard references"; specific sources not recoverable]
- The "3-machine-type test" validation rule was formalized and recorded in STANDARDS.md. [basis: CHANGELOG 14:30 — this is indirect artifact evidence, moderate confidence]

## Likely missing

- The agent's actual plan review output (what it found deficient in the existing plan) is unrecoverable.
- The specific research sources consulted during prompt [3] are unknown beyond the changelog reference to RS274/NGC, grblHAL, LinuxCNC.
- Any plan draft shown to the user during 05:44–06:25 was not committed — only the later session's execution (a02ca962) left artifacts.
- The `/resume` at 09:14 on 2026-04-13 may indicate this session was reopened before session a02ca962 ran; the purpose of that re-open is unrecoverable.
