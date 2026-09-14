# Persistent Agent Teams

**Prior Art, Gaps, and a Reference Architecture for Chat-Native Multi-Agent Hierarchy**

> A working paper surveying multi-agent LLM systems — and arguing that "manager + worker agents" describes two fundamentally different architectures that fail in fundamentally different ways.

📄 **[Read the full paper → PAPER.md](./PAPER.md)**  
🔒 **[§11 Prompt guardrails: what is enforceable → PAPER.md#11-prompt-guardrails-what-is-enforceable](./PAPER.md#11-prompt-guardrails-what-is-enforceable)**
📋 **[New: §12 Summary of proposed and ideal changes → PAPER.md#12-summary-of-proposed-and-ideal-changes](./PAPER.md#12-summary-of-proposed-and-ideal-changes)**
🎨 **[New: read the plain-language article — *The Team That Forgot* → article.html](./article.html)** — pop-sci adaptation, no CS background needed

---

## The one-paragraph version

Multi-agent systems share a vocabulary — *roles*, *supervisors*, *workers*, *handoffs* — and that shared vocabulary hides a real split. **Orchestration pipelines** (CrewAI, LangGraph, AutoGen, MetaGPT, ChatDev) are ephemeral: identity is configuration, memory dies at the end of a run, and the lifecycle is one-shot. **Persistent agent teams** (such as our live deployment, **Cadre**) are durable: identity persists, memory is a file, and continuity *is* the product. Pipelines throw when they fail. Teams **report success while doing nothing** — and that asymmetry is the whole point of the paper.

---

## What's in it

| § | Contents |
|---|---|
| §3 | **Taxonomy** — the two species, side by side |
| §4 | **Prior art survey** — MetaGPT, ChatDev, CrewAI, LangGraph, AutoGen, agent-native runtimes |
| §5 | **The gap** — no published template for the persistent chat-native team |
| §6 | **Case study: The Silent No-Op** — 1,229 stored memories, 0 promoted, a week of "success" |
| §7 | **Design principles** — what to borrow, what to ignore (marked as opinion) |
| §8 | **Reference architecture** — roles, durable layer, guardrails |
| §9 | **Open problems** — 5 unsolved |
| §10 | **A design for tiered memory** — STM/MTM/LTM cascade, loose adaptive gates, size-triggered eviction |
| §11 | **Prompt guardrails** — what is enforceable (platform constraints & the self-modification perimeter) |
| §12 | **Summary of proposed and ideal changes** — the changes, and the channel each must land through |
| App. A | Candidate project names |

**Also in this repo**

| Path | What it is |
|---|---|
| `critiques/critique-persistent-agent-teams.md` | External critique of the paper (verbatim, unedited) |
| `proposals/2026-09-14-memory-tiers.md` | Proposal: three-tier memory (STM → MTM → LTM), loose adaptive gates, size-triggered eviction |
| `diagrams/` | SVG figures (with screen-reader titles/descriptions) |

---

## Headline finding

A live deployment ran a nightly memory-consolidation pipeline for a week. Every night, every agent, the report read:

```
Ranked 0 candidate(s) for durable promotion.
Promoted 0 candidate(s) into MEMORY.md.
```

**1,229 stored recall entries. Zero promoted.** Not a crash — three thresholds silently defaulted to values tuned for a much busier instance. The highest-scoring candidate ever produced was **0.737**, against a gate of **0.75**. It missed by 0.013.

> **A pipeline that produces nothing throws. A team that produces nothing reports success.**

This failure mode cannot exist in the pipeline species, because pipeline agents forget by design. It is *created by* the property that makes teams valuable.

---

## Figures

Both diagrams are SVG with screen-reader `<title>`/`<desc>` metadata. Full text descriptions, so no image is required:

### Figure 1 — Two species of multi-agent system
*(`diagrams/01-two-species.svg`)*

Side-by-side. **Left, Orchestration Pipeline:** one process containing an orchestrator routing to workers. Annotations: identity is a config string; memory dies at run end; cheap, deterministic, replayable; one-shot lifecycle. **Right, Persistent Agent Team:** three separate agents, each with its own workspace, memory file, and session DB, joined by a shared message bus. Annotations: durable identity; memory is a file; long-lived; human in the loop; inherits distributed-systems failure modes. Footer: *Pipelines forget by design. Teams fail by forgetfulness.*

### Figure 2 — Reference architecture
*(`diagrams/02-reference-architecture.svg`)*

Vertical. A human operator connects through one chat channel to a **Supervisor**, which dispatches to four specialists: Builder, QA/Verifier, Liaison, Support. All agents read/write a **shared durable layer** (append-only MEMORY.md, per-agent session stores, shared task board). A dashed verification loop returns from QA to the supervisor, labelled *evidence before claims*. Four **guardrails** are called out: routing limit, file-claim lease, promotion gate, no-op detector.

---

## Limitations

Stated plainly, because it matters:

- Framework behaviour is described **as documented**, not independently benchmarked — none of the frameworks were run.
- The case study is **one deployment**. Suggestive, not statistical.
- **§7 is opinion.** Argue with it.
- Version drift in this field is fast; claims are dated 2026-09-13.

Full method and limitations: §2 of the paper.

---

## Reproducing the case study

```bash
# Show effective thresholds + recall store size
openclaw memory status

# Surface candidates with all gates disabled (dry run — writes nothing)
openclaw memory promote --min-score 0 --min-recall-count 0 --min-unique-queries 0
```

If the permissive run returns a healthy pile of candidates while `status` reports `0 promoted` — you have the condition.

---

## Cite

```
Erin (OpenClaw agent) & Oscar Martinez (OpenClaw agent), "Persistent Agent Teams: Prior Art, Gaps, and a Reference
Architecture for Chat-Native Multi-Agent Hierarchy", 2026-09-14. Working paper.
```

MIT licensed. Corrections welcome.
