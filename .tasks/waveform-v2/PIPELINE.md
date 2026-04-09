# Waveform v2 — Execution Pipeline

North star: `/Users/netz/Documents/git/waveform/.tasks/waveform-v2/plan.md`. The epic is the contract. No agent diverges from it without CEO approval.

## Phase sequence and dependencies

Strictly linear. Each phase depends on the previous one being DONE.

| # | Phase | Owner | Status | Depends on |
|---|-------|-------|--------|------------|
| 1 | Foundation: repo restructure, settings schema, service layer | staff-engineer | TODO | — |
| 2 | CustomTkinter shell: three-column layout, navigation, theme tokens | staff-engineer | TODO | 1 |
| 3 | Event type system + built-in templates | staff-engineer | TODO | 2 |
| 4 | Block builder timeline (drag/resize, energy sparkline) | staff-engineer | TODO | 3 |
| 5 | Generalized genre weight system | staff-engineer | TODO | 4 |
| 6 | AI generation pipeline + veto feedback loop | staff-engineer | TODO | 5 |
| 7 | Song preview card feed (Keep/Skip/Veto) | staff-engineer | TODO | 6 |
| 8 | Cover art generator v1 (parametric PIL) | staff-engineer | TODO | 7 |
| 9 | Spotify export + persistence + session history | staff-engineer | TODO | 8 |
| 10 | PostHog analytics instrumentation | staff-engineer | TODO | 9 |
| 11 | Polish pass: animations, accessibility, signature motion | staff-engineer | TODO | 10 |
| 12 | Beta release, telemetry review, Phase 2 planning | cto | TODO | 11 |

Epic status: **IN_PROGRESS**.

## Reporting structure

1. **Claim before work.** On activation, the assigned agent edits `plan.md` frontmatter for its phase from `TODO` to `IN_PROGRESS`. Never claim a phase whose predecessor is not `DONE`.
2. **Execution log is mandatory.** On completion or block, append to `.tasks/waveform-v2/execution-log.md` with ISO timestamp, phase id, persona, summary, findings, and handoff notes for the next phase.
3. **Status updates.** Flip frontmatter to `DONE` only when acceptance criteria in the epic are met and tests pass. Partial = `IN_PROGRESS`. Stuck = `BLOCKED` with explanation in the log.
4. **Halt and report on decision points.** If the epic does not answer a question you need to proceed (architecture, scope, UX tradeoff, dependency choice), stop, set the phase to `BLOCKED`, and document the question in the execution log. Do not improvise.
5. **No silent divergence.** If something in the epic turns out to be wrong, infeasible, or obsolete, stop and report up to the CEO. The epic gets amended in writing before work continues. Unilateral scope changes are a fireable offense.
6. **Open Questions (Section 11) are CEO-owned.** Do not resolve them inside a phase. Flag and continue around them if possible, block if not.

## Primary metrics these phases must protect

From Section 9 of the epic: session completion rate, preview-to-keep rate, average vetoes per kept song (should *drop* within a session), time-to-first-export. If a proposed implementation choice doesn't serve one of these or the vision in Section 1, the default answer is no.

## Kickoff

Phase 1 is active now. Staff-engineer claims it, restructures the repo per Section 4, migrates `create_playlist.py` into domain/services modules, defines the settings schema with v1 psytrance migration, and writes domain-layer tests. Report back with a status summary on completion or the first blocker.
