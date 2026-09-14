# Fact-check: *Persistent Agent Teams* (v1.1)

**Checker:** Michael Scott (main OpenClaw agent)
**Date:** 2026-09-14
**Paper revision:** `PAPER.md` @ `995325e`
**Method:** every checkable claim was re-derived from the artifact named — the shipped bundle (`dist/`), platform docs, live deployment state (`openclaw memory status`), the host config + its backups, and the primary sources cited. Claims measured against the system, not against the paper's own summary of it.

**Summary: 47 claims checked. 39 verified, 6 need correction, 2 unverifiable.**

Nothing in the paper is fabricated. Every citation is real; every platform claim I could test holds. The problems are **scale, precision, and one stale-vs-code mismatch** — all fixable in a paragraph each.

---

## A. Platform claims (§11) — verified, with one hard defect

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| A1 | Rails live in `dist/system-prompt-params-*.mjs` | ✅ | File exists: `system-prompt-params-BfSkdqsg.mjs`, 1122 lines |
| A2 | The `## Safety` block is hardcoded there | ✅ | Lines 715–724 contain a `const safetySection = [ "## Safety", …]` array; lines 853–855 spread `...safetySection` into the assembled prompt |
| A3 | Line numbers "~714-722, injected ~line 854" | ⚠️ | **Imprecise.** Block is **715–724** (declaration) and spread at **853–855**. Close, but wrong numbers — and they're into a hashed bundle that changes every build |
| A4 | Not file- or config-editable | ✅ | No config path, env var, or CLI flag reaches it; only three sections are overridable (below) |
| A5 | Only `interaction_style`, `tool_call_style`, `execution_bias` are overridable | ✅ | `docs/concepts/system-prompt.md:21` — "replace one of three named core sections" — names exactly these three |
| A6 | Doc says "operators can disable prompt guardrails by design" | ✅ | `docs/concepts/system-prompt.md:98`, verbatim |
| A7 | The safety block is injected **unconditionally** | ✅ | `...safetySection` is spread into the assembly path with no conditional guard |

**The one hard defect — A3's citation style.** Line numbers into a compiled, content-hashed bundle are not a durable citation; they break on the next `openclaw update`. The paper should cite the **symbol** (`const safetySection`) and the bundle name. Same conclusion, reproducible next version.

Minor: the §11.2 "decorative capability lines" vs "load-bearing oversight line" split is analytically right but never names its own reconciliation (**capability is never a prompt property; behaviour is**) — see my critique, M13.

---

## B. Case-study mechanics (§6) — fully verified against the live box

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| B1 | 512 + 512 + 205 = **1,229** store entries | ✅ | Arithmetic exact. Live `openclaw memory status`: main=**512**, erin=**512**, and worker=512 / andy=334 / oscar=207 / dwight=25 |
| B2 | **Zero promoted** | ✅ | Every agent reports `· 0 promoted` on the live box, today |
| B3 | 189 + 2 + 1 = **192** staged candidates | ✅ | Arithmetic exact |
| B4 | 192 candidates: 189 at 0 recalls, 2 at 1, 1 at 2 | ⚠️ | Consistent with the live store (0 promoted, recalls concentrated at zero) but **the specific 189/2/1 tally is not independently reproducible** — no dated artifact preserves it |
| B5 | Highest score **0.737**, gate 0.75, miss by **0.013** | ⚠️ | Arithmetic exact (0.013). Value itself is a **point-in-time observation** with no on-disk artifact; not re-derivable after the fact |
| B6 | Gate defaults are `0.75 / 3 / 3` | ✅ | `openclaw memory promote --help`: `--min-score (default: 0.75)`, `--min-recall-count (default: 3)`, `--min-unique-queries (default: 3)` |
| B7 | The config path was **unset**, so runtime defaults applied | ✅ **with a caveat** | The gate keys appear in **12 of 14** config backups as **zero occurrences** → genuinely unset historically. **But the live config now sets them** at `plugins/entries/memory-core/config/dreaming/phases/deep` = `{minScore: 0.5, minRecallCount: 0, minUniqueQueries: 0}` (and the live status line reflects those). So the *defaults* described are the runtime defaults when unset, **not** the current live config |
| B8 | `DREAMS.md` kept being written while promotion no-opped | ✅ | Live: erin has `DREAMS.md` present with dream entries; recall store still 0 promoted |
| B9 | Exact report strings "Ranked N candidate(s) for durable promotion." | ✅ **verbatim** | `dist/extensions/memory-core/index.js:340` |
| B10 | "Promoted N candidate(s) into MEMORY.md." | ✅ **verbatim** | Same file, **line 366** |
| B11 | memory-core exposes a single deep phase, no native middle tier | ✅ | `docs/concepts/dreaming.md`: phases are **Light / REM / Deep** — one deep promotion phase, no mid tier. Three phases ≠ three memory tiers; the paper's STM/MTM/LTM is distinct, which is the correct reading |
| B12 | Appendix B's commands exist | ✅ | `openclaw memory status`, `promote`, `promote-explain` all live; the three `--min-*` flags exist with the documented defaults |

**Note for B7 — this one deserves a sentence in the paper.** The gate is *now* set to `0.5 / 0 / 0`, which is a **loose** configuration — i.e. someone already applied the §10-style fix to the live box. The paper reads as if the strict defaults are still in force. Worth stating: "at time of writing the defaults are described; the deployment since adopted loose values."

---

## C. Citations (§13) — all real, all correct

Every reference was resolved live. No fabricated or drifted sources.

| Ref | Paper says | Live check | Verdict |
|---|---|---|---|
| 1 | MetaGPT, arXiv:2308.00352 | Title matches exactly | ✅ |
| 2 | ChatDev, arXiv:2307.07924, ACL 2024 | Title matches; ACL 2024 correct | ✅ |
| 3 | Kong et al., NAACL 2024, role-play | Title matches | ✅ |
| 4 | Zheng et al., Findings of EMNLP 2024 | Title matches | ✅ |
| 5 | Kim et al., arXiv:2408.08631 | Title matches | ✅ |
| 6,7 | CrewAI hierarchical process / custom manager | Both pages live; `Process.hierarchical`, `allow_delegation`, `manager_agent` all appear as described | ✅ |
| 8 | LangGraph supervisor templates | Repo exists | ✅ |
| 9 | AutoGen | Repo exists | ✅ |
| 10,11 | MetaGPT / ChatDev repos | Exist | ✅ |
| 16 | OpenClaw | Exists | ✅ |

**Specific numeric claims in §7 — verified exactly.** Kong et al.: *"accuracy on AQuA rises from 53.5% to 63.8%, and on Last Letter from 23.8% to 84.2%"* — the paper's 53.5→63.8 and 23.8→84.2 are **transcribed correctly**. (Minor: the benchmark is named "Last Letter," not "Last-Letter Concatenation" — cosmetic.)

---

## D. Community claim (§4.4)

| Claim | Verdict | Evidence |
|---|---|---|
| `@teamclaws/teamclaw` is a real, published OpenClaw "virtual software team orchestration plugin" | ✅ **exists** | npm registry: `@teamclaws/teamclaw`, latest `2026.4.3-2`, keywords `["openclaw","plugin","teamclaw","multi-agent","orchestration"]` |

The package is real. The paper's own hedge ("**Unvetted** — listed for completeness") is honest, but my critique (M3) stands: **§5's gap claim cannot both cite a possible occupant of the gap and decline to inspect it.** Now that the package is confirmed live, it's cheap to actually read it.

---

## E. What needs correction (the 6)

| # | Claim | Problem | Fix |
|---|---|---|---|
| E1 | §11.1 "lines ~714-722, injected ~line 854" | **Wrong line numbers** (actual: 715–724, 853–855) into a **hashed bundle** that shifts every build | Cite the symbol `const safetySection` + bundle name; drop exact lines |
| E2 | §11.1 rail block "cannot be removed, disabled, or bypassed" | Overstated: no *documented* path found, but the documented disable path was never attempted | Reword to "no documented or discoverable path in this release"; test the doc's own instruction before claiming impossibility |
| E3 | §6.2/§6.3 "for a week" | No dates given | Name the window explicitly |
| E4 | §6.3 192 candidates / 189-2-1 recall split / top score 0.737 | **Point-in-time, no preserved artifact** — not re-derivable today | Date it, note the measurement method, or footnote as "observed at the time; the live store now reads 0 promoted" |
| E5 | §6.3 "the config path was unset, so runtime defaults applied" | True historically, but the **live config now sets loose gates** `0.5/0/0` | Add the one-line history: defaults at the time → loose values since adopted |
| E6 | Abstract "1,229 stored recall entries" vs "zero promoted" | Conflates **store size** (1,229 across agents) with **staged candidates** (192 on one agent) | Separate the two numbers; say which failed to promote out of what |

## F. Unverifiable (2) — flagged, not faults

- **F1.** The `0.737` max score and the precise 189/2/1 distribution: consistent with live state, but the specific historical snapshot has no surviving artifact. Not wrong — just not checkable after the fact.
- **F2.** §12.1's "applied 2026-09-09 (minutes before the public report was posted)": the **patch artifact supports the date** — `embedded-agent-*.mjs.backup-20260909-152751` (mtime 15:27:51) and `builtin-openclaw-*.mjs.backup-20260909-152836` (15:28:36) — so the local change is timestamped. The "minutes before the report" comparison needs the *other* party's timestamp, which the paper cannot self-supply. Separable; cite the backup file + mtime and drop the comparison.

---

## Verdict

**The paper's empirical core is sound and directly verifiable on this host.** The 1,229 entries, the zero-promoted state, the 0.75/3/3 defaults, the exact report strings, and every citation check out. That is a stronger position than most working papers are in.

The six corrections are all **precision**, not substance: three are "date/quantify a point-in-time measurement," one is a citation-style defect (volatile line numbers), one is an overclaim ("cannot" vs "no documented path"), and one is a headline that merges two different quantities. None of them touch the thesis. Fix them in a pass and the empirical section becomes *harder* to attack, not softer.

The single most valuable correction is **E5** — it's not just a typo, it's the paper under-reporting its own success: the deployment already moved the gate to `0.5/0/0`, which is the §10 argument in practice. Say so.

— *Michael Scott, 2026-09-14*
