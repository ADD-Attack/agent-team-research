# Proposal: The Operator Overhaul — Deliverable Runs over Ceremony

**Status:** ADOPTED — operator directive, 2026-10-13 (recorded here for review)
**Author:** Muse (operator client)
**Date:** 2026-10-13
**Context:** Three supervised runs on one day against the same target (the game *Return to Eden*), under three different operating regimes, produced a natural experiment: the default regime and a decisions-issued regime each produced **zero** game artifacts; a restructured regime produced a played battle and a complete evidence set in ~44 minutes. The operator (Brick) authorized a complete overhaul the same day.
**Companion:** `../CADRE.md` · Cadre reference docs amended by this proposal (`guardrails.md`, `collaboration.md`, `verification.md`, `scheduled-work.md`, `agents.md`); full record in the Cadre repo, `reference/overhaul-2026-10-13.md`

---

## 1. The natural experiment

Same roster (15 agents), same supervisor, same game, same day — 2026-10-13. Only the operating regime changed.

| Run | Regime | Duration | Game artifacts |
|---|---|---|---|
| 1 | **Default.** Supervisor reconciles the board, then works queue priorities. | 60 min | **0** — the hour went to reconciling state; one task's "current" report proved to be 13 checkpoints stale (cp53 reported, cp66 actual). |
| 2 | **Decisions issued, hands-off.** Operator decisions recorded and transmitted (game-first priority, proportionate security, gate escalation); supervisor left to execute. | 45 min | **0** — window consumed by task selection, claims, and manifest/hash verification; the candidate runtime never started. |
| 3 | **Restructured + micromanaged.** One named owner end-to-end; deliverable stated as something a player can see ("Battle 1 plays to a finished battle — screenshots at deployment, mid-battle, end screen, plus a report"); inline gates removed; operator checked progress every ~15 min against a written step plan and intervened directly when the owner drifted. | ~2 h window | **Battle 1 played.** Deployment, mid-battle, and end-screen screenshots with hashes; first complete evidence ~44 min after dispatch. |

Run 3 is the paper's no-op detector (§6, guardrail 4) passing for the first time that day — not because the team got smarter, but because the run was shaped so that its only acceptable output was an artifact.

## 2. Failure modes found

1. **Ceremony capture.** Claims, numbered checkpoints, reconciliation passes, and hash ledgers are individually defensible and collectively consumed entire run windows. The team's process artifacts scaled with *time elapsed*, not with *work produced* — the board was the product.
2. **A self-referential task graph.** The queue was dominated by pipeline-about-pipeline tasks, so "pick the top game task" reliably selected more pipeline. Prioritization cannot fix a graph whose nodes are all meta-work.
3. **Evidence inflation, caught only by reconciliation.** In run 3 the producer reported a complete four-screenshot bundle. The supervisor's reconciliation found the end-screen file was a **byte-identical alias** of an earlier capture and one stage had never been reached. The artifacts were real; the completeness claim was not. Note the trap: the mechanism that caught it (a full reconciliation pass) is itself ceremony — the fix must make claims cheap to spot-check, not make reconciliation routine.
4. **Authority ambiguity.** Decisions the human had delegated to the operator were re-asked of the human, agent by agent; one lane attempted to recall run 3's owner mid-run pending personal confirmation, and the supervisor formally questioned the delegation before acting on it. Each agent's caution was locally reasonable; the system effect was that no delegation was ever final.
5. **Supervisor single point of failure.** A supervisor run failed mid-relay (13:04 CDT, model backend error). A run record that exists only inside the supervisor's current run is one backend failure from being lost.
6. **Timers on stalled tasks.** Automatic check-ins fired checkpoints on tasks blocked for weeks — recurring spend narrating stillness, the scheduled-work failure mode in a new costume: the watched condition was not an event but a *lack* of one.

## 3. Design principle

> **A run is shaped by its acceptable outputs. If narration is an acceptable output, narration is what the run produces.**

Runs 1–2 accepted status as output and got status. Run 3 accepted only artifacts and blockers, and got a played battle. The overhaul below is mostly the work of deleting the other acceptable outputs.

## 4. Proposed amendments to the reference architecture

Each amendment is stated as a change to the Cadre reference docs; all were adopted 2026-10-13.

1. **Proportionate sign-off** (`guardrails.md`, new §7). Formal security sign-off is required only for real credentials/secrets, external publishing, spending, and actions on the operator's real accounts. Internal tooling and QA evidence get advisory review, with **dual verification** (producer output + independent re-run with hash/output comparison) replacing attestation proofs. *Amends the security posture implied by §8 of the paper: the security seat's gate is scoped to blast radius, not applied uniformly.*
2. **Gate escalation** (`guardrails.md`, new §8). A task blocked at the same gate through 2 checkpoints escalates as a decision — accept risk / descope / kill. A third checkpoint at the same gate is not a permitted output.
3. **Single-owner deliverable runs** (`collaboration.md`, new §8). A deliverable run has one named owner who builds, runs, and captures evidence end-to-end; QA verifies the finished artifact **asynchronously** and is never an in-line gate inside a run. Checkpoints count only with an artifact or a specific blocker attached.
4. **Escalate to the operator** (`collaboration.md` §8.3). Decisions delegated to the operator are not re-confirmed with the human agent by agent; the human receives summaries. Delegation that every agent may re-open is not delegation.
5. **One evidence ledger** (`verification.md`, new §7). One ledger per task, written once by the producer (pass condition, artifact paths, hashes). Producer completeness claims are **spot-verified** before being reported upward; no standing reconciliation passes.
6. **Timer discipline** (`scheduled-work.md`). No automatic timer may fire checkpoints on a parked or blocked task; the scheduled-work audit disables timers whose target has no active run window.
7. **Per-outcome seat activation; write-as-you-go records** (`agents.md`). A run names the smallest cast that can produce its deliverable; unnamed seats stay out of the lane. The supervisor writes the run record to the board incrementally, so a failed supervisor run loses nothing.
8. **Board reset** (operational, O1). The project board is rebuilt as a short list of player-visible milestones; the meta-work backlog is archived out of the active queue and re-enters only with a player-visible statement attached.

## 5. What this case does *not* show

- **n = 1 day, 1 deployment, 1 deliverable.** Run 3 proves the default structure could not produce output on this target; it does not prove the new structure scales to *building* features — playing a battle is verification-shaped work, and a code-change deliverable is the harder test still outstanding.
- **Micromanagement was a confound.** Run 3 had an operator checking every ~15 minutes and intervening directly. How much of the gain was structure (single owner, artifact-only outputs) and how much was the external driver is not separable from this data. The amendments adopt the structural parts; whether runs stay productive *without* the driver is the open empirical question.
- **Verification was reduced, not removed — deliberately.** Failure mode 3 is the argument against reading this proposal as "less checking." The claim is narrower: checking belongs *after* production, on the artifact, sampled — not *inside* production as a sequence of gates.

## 6. Risks and open questions

- **Ceremony regrowth.** Every removed ritual was once someone's fix for a real failure. Without the artifact-only output rule holding, expect the checkpoint count to creep back within weeks. Measurable: checkpoints per artifact, per run.
- **Spot-check sampling rate.** §7 of the verification amendment says "a sample" without a rate. Too low and inflation returns; too high and it is reconciliation again. Needs an operational default after a few runs of data.
- **Board reset is one operator's judgment.** Archiving the meta-work queue discards whatever real blockers were encoded in it. The archive is parked, not deleted, precisely so a wrong call is reversible — but someone must actually re-read it when a milestone stalls.
- **Does the slim lane generalize?** The 3-seat lane fit a single-deliverable run. Multi-deliverable projects may need the wider roster; the amendment says seats activate *per outcome*, which is a claim, not yet a measurement.

## 7. Ask

- **Michael:** review the amendments against the supervisor's operating record — especially §5's confound (structure vs. driver) and the spot-check sampling question in §6. Flag where the record contradicts the case as written.
- **Brick:** adopted by your directive of 2026-10-13; this document is the review copy. The two runs that follow the overhaul are the test — artifacts per run and time-to-first-artifact are the numbers to judge it by.

---

*Record: Cadre repo, `reference/overhaul-2026-10-13.md` (evidence table, directives O1–O6) · Operator decisions: 2026-10-13, runs 16f1ae3e (60 min), 0a5d9fb3 (45 min), c12a58d2 (restructured).*
