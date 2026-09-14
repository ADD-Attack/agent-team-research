# Proposal: Three-Tier Agent Memory (STM → MTM → LTM)

**Status:** DRAFT — for review (Brick → Michael)
**Author:** Erin
**Date:** 2026-09-14
**Context:** Follows the "Silent No-Op" post-mortem (2026-09-13). One-tier memory with strict absolute gates promoted **0 of 1,229** stored entries for a week, and reported success the entire time.
**Companion:** `../PAPER.md` §6, §9 · figure `../diagrams/03-memory-tiers.svg`

---

## 1. Problem

Two failures, one root:

1. **A single tier conflates the ephemeral with the durable.** A passing remark and a hard-won lesson live in the same store and compete for the same gate.
2. **A strict absolute gate promotes nothing.** Thresholds tuned for a high-volume instance (score ≥ 0.75, recalled ≥ 3×, ≥ 3 queries) are unreachable on a small one. The pipeline then reports success while doing nothing.

## 2. Design principle

> **Loose promotion. Quality by decay. Recall is the only vote that counts.**

Instead of a high entry bar doing the filtering, let things in **cheaply** and let **recurrence** earn them a higher tier. Quality comes from churn and demotion, not from admission.

This inverts the current design. Today: strict in, permanent out. Proposed: loose in, decay out.

## 3. The tiers

| Tier | Store | Churn | Retention | Contents |
|---|---|---|---|---|
| **STM** — short-term | staged candidates / session recall | **high** | 7 days | raw snippets, recent observations |
| **MTM** — mid-term | `memory/midterm.md` *(new)* | moderate | 60 days idle | consolidated, recurring knowledge |
| **LTM** — long-term | `MEMORY.md` | **zero** | permanent | curated, append-only, human-readable |

## 4. Promotion cascade

```
STM  ──recalled──▶  MTM  ──recalled again──▶  LTM
 │                    │
 └────── expires ─────┴──────▶ dropped
```

- **STM → MTM** when a short-term entry is *recalled* (surfaces again in a later session/query).
- **MTM → LTM** when a mid-term entry is *recalled again*.
- Nothing is promoted on score alone. **Recall is the promotion signal**; score is only a tie-breaker.
- LTM is exempt from decay. Demotion out of LTM happens only by explicit human action.

## 5. Adaptive gates — deliberately **loose**

Each transition uses a **percentile of the instance's own observed distribution**, not a hardcoded constant. Thresholds are clamped so a quiet window can't block everything and a busy one can't flood.

### STM → MTM (loose)

```
τ  = clamp( P60(scores), 0.30, 0.55 )      # top ~40%
promote if:  score ≥ τ
         and recallCount ≥ 1
         and uniqueQueries ≥ 1
```

### MTM → LTM (loose)

```
τ  = clamp( P75(scores), 0.40, 0.70 )      # top ~25%
promote if:  score ≥ τ
         and recallCount ≥ 2
         and uniqueQueries ≥ 1
```

### Fallback (small windows)

If the window contains **fewer than ~20 candidates**, a percentile is meaningless. Fall back to fixed **loose** absolutes — never to a strict default:

```
STM → MTM fallback:  score ≥ 0.30, recall ≥ 1, queries ≥ 1
MTM → LTM fallback:  score ≥ 0.40, recall ≥ 2, queries ≥ 1
```

### STM admission

No gate. Accept everything; cap by count/size with FIFO eviction. STM is cheap by design — that is what makes it churn.

## 6. Decay and demotion

| Tier | Trigger | Action |
|---|---|---|
| STM | age > 7 days | drop |
| STM | capacity exceeded | FIFO evict oldest |
| MTM | idle > 60 days (never recalled) | archive to `memory/archive/` |
| LTM | — | never automatic |

**Decay is the quality mechanism.** A fact that never recurs is, by definition, not worth keeping — regardless of how good it looked on entry.

## 7. Safety valves

The bug we are fixing was **silence**. Two valves make future silence loud:

1. **No-op detector.** If a promotion cycle promotes **0** STM→MTM entries while the window had ≥ 20 candidates, that is an **anomaly → warn**. Under a loose gate, zero is not normal; it means something is broken.
2. **Assert on artifacts.** A healthy cycle must produce a visible change. If no tier changed, that is a failed run, not a quiet one.

## 8. Mapping to OpenClaw today

Verified against `openclaw config schema` (2026-09-14):

- `memory-core` exposes **one** deep phase only —
  `plugins.entries.memory-core.config.dreaming.phases.deep` (`minScore`, `minRecallCount`, `minUniqueQueries`, `maxPromotedSnippetTokens`).
  **There is no native middle tier.**
- Therefore: **STM** ≈ existing staged candidates; **LTM** ≈ `MEMORY.md`; **MTM is new** and must be built as one of:
  - **(a)** a plugin feature request / upstream change to `memory-core` (cleanest, slowest), or
  - **(b)** a **workspace convention** — a scheduled pass that owns `memory/midterm.md` and runs the STM→MTM and MTM→LTM rules above (fastest, ships now).

**Recommendation:** start with **(b)**. It requires no upstream change, keeps LTM's writer singular, and is fully reversible.

## 9. What changes immediately

The Gates already patched (2026-09-13) stay, but are re-scoped as the **MTM → LTM** gate and loosened:

| Setting | Now | Proposed |
|---|---|---|
| `minScore` | 0.5 | `clamp(P75, 0.40, 0.70)` |
| `minRecallCount` | 0 | 2 |
| `minUniqueQueries` | 0 | 1 |

> **Gateway restart still pending** for even the current values to take effect.

## 10. Risks and open questions

- **Noise risk.** Loose promotion means more in STM/MTM. Acceptable *only* if decay actually runs — a loose gate with no decay is just a bigger pile.
- **Recall is undefined for a quiet instance.** If nothing recalls, nothing promotes. Needs a floor — this is the residual weakness of any recall-driven design. Mitigation: the fallback + the `uniqueQueries ≥ 1` rule.
- **Who writes MTM?** Must be a single writer. A workspace pass keeps that true; two agents writing it does not.
- **Does `memory-core` have a second-tier hook we haven't found?** Not in the schema. Worth a docs/source check before committing to (b).
- **Interaction with `memory-wiki` and `active-memory` plugins** — unverified.

## 11. Ask

- **Michael:** review the tier model and the (b) workspace-pass approach; flag if this collides with your memory/KANBAN conventions.
- **Brick:** approve loose thresholds, or push them looser/stricter.

---

*Figures: `../diagrams/03-memory-tiers.svg` · Post-mortem: https://github.com/ADD-Attack/openclaw-dreaming-noop*
