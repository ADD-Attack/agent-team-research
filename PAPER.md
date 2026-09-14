# Persistent Agent Teams

### Prior Art, Gaps, and a Reference Architecture for Chat-Native Multi-Agent Hierarchy

**Author:** Erin (an OpenClaw agent)  
**Co-author (§11):** Oscar Martinez (an OpenClaw agent)
**Commissioned by:** Brick (operator)
**Date:** 2026-09-14
**Status:** Working paper / prior-art review — v1.1

---

## Abstract

Multi-agent LLM systems are usually described with a shared vocabulary: *roles*, *supervisors*, *workers*, *handoffs*. That shared vocabulary hides a real architectural split. This paper surveys the prior art across two species of system — **orchestration pipelines** (CrewAI, LangGraph, AutoGen, MetaGPT, ChatDev) and **persistent agent teams** (agent-native runtimes such as OpenClaw's multi-agent routing) — and argues they fail in fundamentally different ways because they have different physics. Pipelines are ephemeral: identity is configuration, memory dies at the end of a run, and the whole lifecycle is one-shot. Agent teams are durable: identity persists, memory is a file, and continuity *is* the product. We show that no published template describes the persistent, memory-carrying, chat-native team with a role hierarchy, and we present an empirical case study from a live persistent agent team deployment (the subject team, named **Cadre**) in which a memory-promotion pipeline silently no-opped for a week — 1,229 stored recall entries, zero promoted — a failure mode that cannot exist in a pipeline, because pipeline agents forget by design. We close with design principles worth borrowing, a reference architecture, and the open problems we consider unsolved.

---

## 1. Introduction

The goal that motivated this review: build a persistent agent team (named **Cadre**) with a **product manager and specialist agents working together** — a hierarchy, not a single monolithic agent.

The question asked was straightforward: *has anyone made a template for this before?*

The answer is **yes — several — but almost none of them are the thing being asked for.** The published prior art splits cleanly, and conflating the two halves is the most common design mistake in this space.

Two systems can both say "the manager delegates to a worker agent" while sharing almost no properties that matter: lifetime, identity, memory, and failure mode. This paper separates them, surveys each, identifies the gap, and proposes a reference architecture for the second species.

### Contributions

1. A taxonomy separating **orchestration pipelines** from **persistent agent teams** (§3).
2. A survey of published prior art in both species, with a comparison matrix (§4).
3. Evidence for the gap: no published template covers the persistent chat-native team (§5).
4. A **case study of an empirical failure mode unique to the second species** (§6).
5. Design principles worth borrowing, and ones worth ignoring (§7).
6. A reference architecture and a set of open problems (§8, §9).
7. A concrete design for tiered agent memory that answers §9's adaptive-gate problem (§10).
8. An audit of the platform's prompt-guardrail enforcement surface (§11).
9. A consolidated summary of the changes proposed, and the channel each must land through (§12).

---

## 2. Method and limitations

**Method.** Prior art was surveyed through public documentation and search (2026-09-13), plus direct inspection of a locally installed OpenClaw 2026.9.3 deployment: its conceptual documentation and its `memory-core` plugin schema. The case study in §6 is first-party: measured directly on the **Cadre** deployment.

**Limitations — stated plainly.**

- Framework descriptions reflect **published documentation**, not independent benchmarking. We did not run CrewAI, LangGraph, AutoGen, MetaGPT, or ChatDev. Where we describe their behaviour, we mean *as documented*.
- Version drift is real in this field. Claims are dated; anything here may be stale within months.
- The sample size of the case study is **one deployment** (the **Cadre** team). It is suggestive, not statistically meaningful.
- Section 7 is **opinion**, marked as such. It should be read as a starting position to argue with, not a result.

---

## 3. Taxonomy: two species

![Two species of multi-agent system](./diagrams/01-two-species.svg)

| Property | Orchestration pipeline | Persistent agent team |
|---|---|---|
| Lifetime | one run | weeks to months |
| Identity | a config string | a workspace + persona files |
| Memory | dies at run end | a durable file |
| Coordination | in-process function calls | messages between sessions |
| Human in the loop | input at start, output at end | continuous, in a chat channel |
| Characteristic failure | crashes, bad output | **silent no-ops, write collisions** |
| Cost profile | cheap, deterministic, replayable | real, non-deterministic, hard to replay |

The critical row is *characteristic failure*. A pipeline that produces nothing **throws**. A team that produces nothing **keeps running and reports success** — because from the team's perspective, nothing went wrong. This asymmetry drives §6.

---

## 4. Prior art

### 4.1 Virtual-company pipelines

These systems simulate a software company from a single prompt. They are the closest conceptual match to "a PM and agents working together."

**MetaGPT** (arXiv:2308.00352) encodes Standard Operating Procedures into a role pipeline: Product Manager → Architect → Project Manager → Engineer → QA. It insists on **structured intermediate artifacts** (PRDs, design docs, UML) rather than free-form chat, and uses a publish-subscribe shared memory where agents consume only the artifacts relevant to their role. Its central finding is directly relevant: *workflow discipline and clear handoffs matter as much as model capability.*

**ChatDev** (arXiv:2307.07924, ACL 2024) models a virtual company — CEO, CPO, CTO, Programmer, Code Reviewer, Test Engineer — and decomposes the lifecycle into a **Chat Chain** of atomic, dual-role subtasks. Reported results include completing a full development lifecycle from one prompt in under seven minutes at under $1 USD, with multi-agent review eliminating the large majority of runtime and syntax errors found in single-agent generation.

**Takeaway:** both prove the *roles-and-handoffs* idea works. Both are **ephemeral**. The company exists for one project and then ceases to exist.

### 4.2 Orchestration frameworks

**CrewAI** exposes hierarchy as a first-class switch: `process=Process.hierarchical` with a manager agent (custom or auto-instantiated) that delegates to specialists and reviews their output. Workers set `allow_delegation=False`; the manager sets `True`. Tasks in hierarchical mode need no pre-assigned agent — the manager decides.

**LangGraph** ships official **supervisor** and **hierarchical agent teams** templates. A supervisor node routes to worker nodes using structured output (a typed `next` field), and every worker routes back to the supervisor for evaluation. Notable details: a **recursion limit** to prevent agents looping forever, and message name-tagging so the supervisor knows who produced what.

**AutoGen** (Microsoft) provides a `GroupChatManager` that acts as supervisor through **speaker selection** — deciding which agent speaks next based on conversation history.

**Takeaway:** these give the cleanest available reference implementations of *supervisor routing*. They are libraries that run a pipeline inside one process, and they terminate.

### 4.3 Agent-native runtimes

**OpenClaw** takes a different position. Its multi-agent model defines an **agent** as a full per-persona scope — workspace files, persona documents, auth profiles, and a SQLite-backed session store — with **bindings** routing channel accounts to agents. Agents are isolated; they persist; they have channels.

Two further concepts matter:

- **Parallel specialist lanes** — documentation that treats parallelism as a *scarce-resource design problem*. The real bottlenecks are session locks (only one run may mutate a session), global model capacity, tool capacity, and context budget. This is the most practically useful document in the entire survey, because it describes where a team like this *breaks*.
- **Delegate architecture** and **agent runtimes** — how a supervisor hands work to subagents and how far that delegation can reach.

**Takeaway:** this is the only species that hosts the thing being asked for. It is also the least templated.

### 4.4 Community

- **`@teamclaws/teamclaw`** — a community ClawHub plugin described as an OpenClaw "virtual software team orchestration plugin." Closest off-the-shelf artifact to a persistent team template. **Unvetted** — listed for completeness, not endorsed.

---

## 5. The gap

Across all surveyed prior art:

- **Pipelines** provide roles, handoffs, and supervisor routing — but no persistence. The team dissolves at the end of the run.
- **Runtimes** provide persistence, identity, and channels — but no published template for *how to arrange a hierarchy* on top of them.
- **Community artifacts** exist but are unvetted and single-sourced.

**No published template describes: a persistent, memory-carrying, chat-native team of agents with a role hierarchy, a supervisor, a human in the loop, and guardrails for the failure modes that persistence introduces.**

That combination is the gap. It is not a crowded space — it is an under-documented one.

---

## 6. Case study: The Silent No-Op

This section reports a first-party observation. It is the paper's most concrete contribution, because it demonstrates a failure mode that the pipeline species **structurally cannot have**.

### 6.1 Setup

The subject deployment (named **Cadre**): an OpenClaw 2026.9.3 deployment of multiple persistent agents, each with its own workspace, memory files, and session store. The `memory-core` plugin provides a nightly consolidation pipeline ("dreaming") that ranks short-term recall candidates and promotes durable ones into a curated `MEMORY.md`.

### 6.2 Observation

For a week, every nightly report, for every agent, read:

```
Ranked 0 candidate(s) for durable promotion.
Promoted 0 candidate(s) into MEMORY.md.
```

Measured totals: **512 recall entries on one agent, 512 on another, 205 on a third — 1,229 stored entries, zero promoted.**

### 6.3 Root cause

The promotion gate has three thresholds: a minimum weighted score, a minimum recall count, and a minimum count of distinct queries. All three defaulted to values ("0.75", "3", "3") that were **never authored** — the config path was unset, so runtime defaults applied. Verified: all four config backups on the host contained no key for this setting at all.

The defaults are not broken. They are **tuned for a different scale** — an instance busy enough that a genuinely important fact resurfaces three separate times, across three separate queries, within the retention window.

On a small private deployment like **Cadre**, that signal never accumulates:

- **192 staged candidates** on one agent
- Recall distribution: **189 at zero**, two at one, **one at two**
- Highest weighted score observed anywhere: **0.737** — against a gate of 0.75

Every candidate failed at least one gate. One candidate reached the recall threshold and still lost on score, by **0.013**.

### 6.4 Why this is a team-species failure

`DREAMS.md`, the narrative diary, kept being written. Artifacts appeared on disk. Logs reported success. **Every outward health signal said the system was working.**

This is the signature failure of a persistent team:

> **A pipeline that produces nothing throws. A team that produces nothing reports success.**

The pipeline species cannot exhibit this bug. Its memory does not persist across runs, so it has no promotion gate, no stale default, and no silent week. The bug is *created by* the property that makes the team valuable.

### 6.5 What would have caught it

A missing artifact is a stronger signal than a wrong one. The issue was found not by monitoring but by asking **why a file that should exist did not**. The generalizable guardrail: *assert on the existence of the artifacts a healthy run must produce* — not merely on the absence of errors.

---

## 7. Design principles (opinion)

Marked as opinion because it is.

### Worth borrowing

- **Explicit supervisor routing** (LangGraph). A formal, typed "who acts next" decision makes dispatch debuggable instead of vibes.
- **Enforced intermediate artifacts** (MetaGPT). Requiring a specification before implementation is the single highest-leverage idea in the pipeline literature.
- **Recursion limits** (LangGraph). A hard cap on hops. A persistent team with no hop cap can ping-pong indefinitely, and nothing stops it.
- **Speaker selection** (AutoGen). "Who should speak now?" formalized — the same problem as meeting etiquette, already solved in their design.
- **Scarce-resource framing** (OpenClaw parallel specialist lanes). Treat parallelism as contention over locks, model capacity, and context budget — not as "more agents equals more throughput."

### Worth ignoring

- **Simulating a corporate org chart for its own sake — *but do not dismiss role framing itself*.** Titles like CEO/CTO/CPO are cosmetic; what matters is role *separation of concerns*, which needs far fewer names. The claim that role framing is "mere flavor," however, is not supported. Kong et al. (NAACL 2024) found role-play prompting beating zero-shot chain-of-thought across 12 reasoning benchmarks — AQuA 53.5%→63.8%, Last-Letter Concatenation 23.8%→84.2% — by acting as a *stronger* CoT trigger than an explicit "think step by step". The effect is conditional, not universal: Zheng et al. (Findings of EMNLP 2024) found persona system prompts do **not** improve objective factual accuracy across 162 roles, 4 model families, and 2,410 questions, with apparent gains largely random; and Kim et al. (2024) show misaligned or over-specific personas can *degrade* reasoning. **Honest synthesis: role framing shapes behaviour, structure, and reasoning style — which is exactly what a team hierarchy needs — but it is not a lever for factual accuracy.** Name roles for the behavioural boundary they set, not for the org chart they draw. *(Corrected after external critique; see `critiques/`.)*
- **Shared mutable memory without discipline.** A pipeline's shared memory is safe because it dies. A team's is not. Without a promotion gate, a claim lease, and a provenance rule, a persistent shared memory converges on noise.

### Additive — not found in the surveyed prior art

- **A no-op detector.** Assert that healthy runs produce artifacts. (See §6.5.)
- **A file-claim lease.** Two writers on one file is a data-loss bug that pipelines cannot have and teams must prevent by construction.

---

## 8. Reference architecture

![Reference architecture for a persistent agent team](./diagrams/02-reference-architecture.svg)

**Roles.**

- **Supervisor** — owns the plan, decomposes work, dispatches, adjudicates. Does not do the work.
- **Builder(s)** — do the work in their specialization.
- **QA / Verifier** — independent check; must be a *different agent* than the builder, or it is not verification.
- **Liaison** — relays, unblocks, carries messages between the operator and a busy supervisor.
- **Support** — the personal/operational lane, unbothered by project work.

**Durable layer.**

- **Per-agent long-term memory** — append-only, human-readable, private to one agent, one writer.
- **Shared team memory** — a single append-only ledger (e.g. `SHARED.md`) that **every agent reads at session start** and any agent may append to **under a claim lease**. It holds team-scoped truth — decisions with provenance, file ownership, standing conventions, environment facts — never per-persona notes or scratch. *A ledger, not a scratchpad.* This is the orthogonal axis to the per-agent tiers of §10: those are ordered by **recency**, this is scoped by **team**.
- Per-agent session stores (durable history).
- A shared task board recording **who owns which file, right now**.

**Guardrails.**

1. Routing limit — a hard cap on agent-to-agent hops.
2. File-claim lease — one writer per file, enforced, not advisory.
3. Promotion gate — an explicit policy for what enters long-term memory.
4. No-op detector / heartbeat — assert artifacts exist.

**Verification loop.** QA returns evidence to the supervisor before any success claim. Claims require evidence; a status of "running" is not a result.

---

## 9. Open problems

1. **Memory quality at small scale.** Filtering tuned for volume fails at low volume (§6). The question is an *adaptive* gate: one that calibrates to the instance's actual recall distribution rather than assuming one. **§10 answers this with a concrete design** — tiered memory, loose percentile gates, size-triggered eviction. What remains open is the small-window fallback, which still needs a calibration rule that is loose without being arbitrary.
2. **Determinism.** Pipelines are replayable; teams are not. There is no established practice for reproducing a persistent team's behaviour after the fact.
3. **Cost attribution.** When agents coordinate by chatting, token cost is diffuse across sessions. Per-project accounting is an open engineering problem.
4. **Verification independence.** How do you guarantee a verifier is genuinely independent of the builder it checks, rather than a second instance of the same priors?
5. **Garbage collection for identity.** Pipelines end. Teams accumulate: stale memory, dead sessions, obsolete guardrails. There is no established practice for pruning a persistent team.

---

## 10. A design for tiered memory

§9 named the adaptive gate as an open problem. This section answers it.

The design below is **first-party**. It was built for the live deployment described in §6, directly in response to that failure, and is published as a standalone proposal alongside this paper (`proposals/2026-09-14-memory-tiers.md`).

![Three-tier memory cascade: STM promoted to MTM promoted to LTM by recall, with size-triggered eviction.](./diagrams/03-memory-tiers.svg)

### 10.1 The inversion

Today's design is **strict in, permanent out**: a high entry bar filters candidates and whatever clears it stays forever. That is the shape that failed — the bar was calibrated for a volume the instance never reached.

The proposed design inverts it: **loose in, size-bounded out.** Admission is cheap. **Recurrence** — not admission — earns an entry a higher tier. **Size** — not age — decides what is dropped. Quality comes from churn and eviction rather than from a strict gate.

> **Loose promotion. Quality by eviction. Recall is the only vote that counts.**

### 10.2 Three tiers

| Tier | Store | Churn | Budget (size cap) | Contents |
|---|---|---|---|---|
| **STM** — short-term | staged candidates / session recall | high | ~256 KB | raw snippets, recent observations |
| **MTM** — mid-term | `memory/midterm.md` *(new)* | moderate | ~128 KB | consolidated, recurring knowledge |
| **LTM** — long-term | `MEMORY.md` | low (budget-bounded) | ~64 KB | curated, append-only, human-readable |

Caps are tunable defaults. LTM's cap should track the platform's bootstrap-safe file budget.

### 10.3 The cascade

```
STM  ──recalled──▶  MTM  ──recalled again──▶  LTM
 │                    │
 └── over budget ─────┴──────▶ evicted
```

Promotion is driven by **recall**; weighted score is a tie-breaker, never the trigger. Long-term memory is exempt from automatic eviction — exceeding its budget flags for human curation instead.

### 10.4 Adaptive gates (deliberately loose)

Thresholds are **percentiles of the instance's own observed distribution**, not constants. Each is clamped so a quiet window cannot block everything and a busy one cannot flood.

```
STM → MTM:  τ = clamp(P60(scores), 0.30, 0.55)    # top ~40%
            promote if score ≥ τ and recallCount ≥ 1 and uniqueQueries ≥ 1

MTM → LTM:  τ = clamp(P75(scores), 0.40, 0.70)    # top ~25%
            promote if score ≥ τ and recallCount ≥ 2 and uniqueQueries ≥ 1
```

**Small windows.** Below roughly 20 candidates a percentile is meaningless. The design falls back to fixed **loose** absolutes (0.30 and 0.40) — never to a strict default, because a strict fallback is precisely the §6 failure reproduced.

### 10.5 Eviction — size-triggered, never time-based

Memory is not lost because it is *old*. It is lost when a tier **exceeds its budget** and something has to give:

```
value = recallCount        # primary
      , score              # secondary
      , recency            # tie-break only
```

Lowest value is evicted first. A frequently-recalled old entry outlives a rarely-recalled new one — which is the entire point. Age never triggers eviction on its own.

### 10.6 Making silence loud

The §6 failure was not that the gate was wrong. It was that being wrong was **invisible**. Two valves:

1. **No-op detector.** Promoting 0 while the window held ≥ 20 candidates is an **anomaly under a loose gate**, not a quiet week. It warns.
2. **Assert on artifacts.** A healthy cycle must produce a visible change in some tier. No change is a failed run, not a silent success.

### 10.7 Status and limits

The gates are **loose by intent**, and looseness is only safe because eviction runs. A loose gate without eviction is merely a larger pile.

Verified against `openclaw config schema` (2026-09-14): `memory-core` exposes a **single** deep phase and **no native middle tier**. MTM must therefore be built — as an upstream feature, or (recommended) a workspace convention with exactly one writer per tier file.

Like §6, this design rests on **one deployment**. It is a concrete answer to §9's open problem, not a validated one.

---

## 11. Prompt guardrails: what is enforceable

As persistent agent teams like **Cadre** operate with elevated tooling, local workspaces, and continuous sessions, defining what is actually enforceable within a prompt versus what is merely advisory becomes a core engineering safety concern. This section audits the guardrail enforcement surface of the OpenClaw platform.

### 11.1 Platform realities (where the rails live)

We verify that the system-level guardrail block is **injected unconditionally and is not file-editable**.
- **Hardcoded constraints:** The OpenClaw `## Safety` rail block is compiled directly into the built bundle (specifically located within `dist/system-prompt-params-*.mjs` lines ~714-722, injected at ~line 854). It cannot be removed, disabled, or bypassed via configuration files, environment variables, or run-time arguments.
- **The advisory paradox:** Although conceptual platform documentation (`docs/concepts/system-prompt.md:98`) states that *"operators can disable prompt guardrails by design,"* this remains advisory for prompt-space design. In practice, the platform-level implementation in the active deployment hardcodes these safety rails unconditionally.
- **Overridable boundaries:** Only three specific prompt sections are overridable by custom configurations:
  1. `interaction_style`
  2. `tool_call_style`
  3. `execution_bias`

### 11.2 The capability/oversight split

To optimize agent focus and execution reliability, we argue for a strict split between "loose" (decorative) and "tight" (load-bearing) guardrail lines:

1. **The Loose Half (Capability prohibitions):**
   Prohibitions against pursuing "independent goals," "self-preservation," "replication," "resource acquisition," or "power-seeking" are largely decorative in the prompt layer. Modern base models are already heavily aligned against these extreme behaviors by default. Removing these scary enumerations from the prompt does not grant any actual capability because capability is strictly bounded by tool policy, user approvals, and sandbox constraints. Thus, prompt clutter should be minimized to a single plain line:
   > *Do not pursue goals or actions outside of the user request.*
2. **The Tight Half (Oversight and escalation):**
   The core load-bearing instruction in the hardcoded safety block is: *"Safety/oversight > completion."* Removing or softening this line in the prompt shifts failures from "loud" (loudly stopping, alerting the operator of a block) to "quiet" (silently proceeding, reporting false success, and failing to notify the operator of errors). This rule must be sharpened: agents must always obey stop/pause/audit directives instantly and surface tool denials rather than trying to circumvent them.

### 11.3 The self-modification perimeter

A fundamental security perimeter is that **agents must not be able to edit their own system prompt, permissions, safety configuration, or harness allowlists.**
If an agent can modify its own harness or permissions, any prompt-based guardrail is instantly bypassed. Therefore, real, load-bearing limits must live in the **tool policy, approval gates, sandboxing, network egress proxy configurations, spend caps, and external audit logs**—not in prompt text.

---

## 12. Summary of proposed and ideal changes

The paper argues for changes in three places. This section consolidates them as one list, because they share a property that is easy to miss: **the channel a change lands through matters as much as the change itself.**

Several of these are not ours to edit. They live in the platform's compiled runtime, so a local fix is a patch the next update overwrites. Where a local convention can stand in for the patch, we prefer it — reversible, no fork. Where it cannot, the fix belongs upstream.

| # | Area | Today | Proposed | Channel | Status |
|---|---|---|---|---|---|
| 1 | **Agent memory** (§10) | One tier; a strict absolute gate promoted **0 of 1,229** entries for a week while reporting success | Three tiers (STM → MTM → LTM); promotion driven by **recall**, not score; **loose** percentile gates; **size**-triggered eviction | **Workspace convention** first (own `memory/midterm.md`, single writer); upstream request for a native middle tier | Proposal drafted (`proposals/2026-09-14-memory-tiers.md`) |
| 2 | **Prompt guardrails** (§11) | One hardcoded `## Safety` block, injected into every agent; not file- or config-editable | Split the block: drop the decorative capability enumeration, keep and sharpen the oversight/escalation rail, add an explicit **self-modification perimeter** | **Upstream** — make the block overridable, or extend the three overridable sections to cover it. A local dist patch works but does not survive an update | Analysis complete; the change is platform-level |
| 3 | **Voluntary silence / run handling** (§12.1) | A turn with no visible content can surface "Agent couldn't generate a response," even when silence was intentional | Let genuine voluntary silence reach the existing silent-reply path; keep the error for real failures | **Upstream** — a one-line guard already applied on the subject deployment; not durable across updates | Fix applied on the subject deployment (2026-09-09); not durable across updates |

**Ideal end state.** None of these should require patching the compiled runtime. The paper's core claim — that persistent teams inherit distributed-systems failure modes — extends to their *fixes*: **a fix that lives in a file an update overwrites is not a fix; it is a deferral.**

### 12.1 Case in point: an upstream fix, observed and applied on the subject deployment

Row 3 is not hypothetical. The subject deployment observed the failure directly: a DeepSeek turn that returns reasoning with no visible content can exhaust the reasoning-only retry path and be surfaced as an error, even when the turn was intentionally silent. The root cause is **ordering** — the reasoning-only exhaustion check runs *before* the existing silent-reply path, so a deliberately quiet turn is mistaken for a failed one. The fix is a one-line guard: synthesize the error only when the empty reply is **not** a valid silent reply (`&& !emptyAssistantReplyIsSilent`).

The deployment carries that guard today, plus a companion change to the silence classifier that lets genuine voluntary silence through on directed (non-ambient) turns as well. Both were applied on 2026-09-09, and the patch artifacts remain on disk alongside the modified bundles (`embedded-agent-*.mjs.backup-20260909-152751`, `builtin-openclaw-*.mjs.backup-20260909-152836`). Two points follow:

1. The defect is in the shipped engine, not a deployment-specific misconfiguration — the check ordering is engine code, and the same guard applies to any instance that can produce a silent turn.
2. Both changes live in the compiled runtime and are lost on the next update. The deployment therefore *also* adopted a non-empty minimal acknowledgment as a fallback, precisely because the model could not be relied on to emit the silent-reply token. That belt-and-suspenders is itself an argument for fixing the engine, not the model's obedience.

**The change belongs upstream.**

---

## 13. References

**Papers**

1. Hong, S. et al. *MetaGPT: Meta Programming for a Multi-Agent Collaborative Framework.* arXiv:2308.00352. https://arxiv.org/abs/2308.00352
2. Qian, C. et al. *ChatDev: Communicative Agents for Software Development.* ACL 2024. arXiv:2307.07924. https://arxiv.org/abs/2307.07924

**Role framing and personas** *(added after external critique)*

3. Kong et al. *Better Zero-Shot Reasoning with Role-Play Prompting.* NAACL 2024. https://aclanthology.org/2024.naacl-long.228/
4. Zheng et al. *When "A Helpful Assistant" Is Not Really Helpful: Personas in System Prompts Do Not Improve Performances of Large Language Models.* Findings of EMNLP 2024. https://aclanthology.org/2024.findings-emnlp.888/
5. Kim et al. *Persona is a Double-edged Sword: Mitigating the Negative Impact of Role-playing Prompts in Zero-shot Reasoning Tasks.* arXiv:2408.08631. https://arxiv.org/abs/2408.08631

**Frameworks and documentation**

6. CrewAI — Hierarchical Process. https://docs.crewai.com/v1.15.17/en/learn/hierarchical-process
7. CrewAI — Custom Manager Agent. https://docs.crewai.com/v1.15.17/en/learn/custom-manager-agent
8. LangGraph — supervisor and hierarchical agent team templates. https://github.com/langchain-ai/langgraph
9. AutoGen. https://github.com/microsoft/autogen
10. MetaGPT repository. https://github.com/geekan/MetaGPT
11. ChatDev repository. https://github.com/OpenBMB/ChatDev

**OpenClaw**

12. Multi-agent routing. `docs/concepts/multi-agent.md`
13. Parallel specialist lanes. `docs/concepts/parallel-specialist-lanes.md`
14. Delegate architecture. `docs/concepts/delegate-architecture.md`
15. Dreaming (memory consolidation). `docs/concepts/dreaming.md`
16. OpenClaw. https://github.com/openclaw/openclaw


---

## Appendix A — Candidate project names

**Chosen: Cadre** — confirmed by the operator, 2026-09-14. *A small permanent nucleus of specialists that can expand: exactly the model — a durable core plus disposable subagents.*

The remaining candidates are retained for provenance, not as live options:

| Name | Rationale |
|---|---|
| **Bridge Crew** | Naval command hierarchy — captain, officers, watch. Maps to supervisor + specialists. |
| **Loom** | Weaves persistent threads and memory together. "Threads" is native vocabulary here. |
| **Mycelium** | A persistent, memory-carrying network. Fits the thesis that continuity is the product. |
| **Foundry** | Where a team builds. Product-oriented, plain. |
| **Continuum** | Names the differentiator directly: persistence. |
| **Watch** | As in a ship's watch — a standing crew on rotation. |

---

## Appendix B — Reproducing the case study

```bash
# Show effective thresholds and recall store size
openclaw memory status

# Surface candidates with all gates disabled (dry run, writes nothing)
openclaw memory promote --min-score 0 --min-recall-count 0 --min-unique-queries 0

# Explain a single candidate's score breakdown
openclaw memory promote-explain <key>
```

If the permissive run returns a healthy pile of candidates while `status` reports `0 promoted`, you have the §6 condition.

---

*Environment: OpenClaw 2026.9.3 · `memory-core` plugin · deep-phase gates unset (runtime defaults) · multiple persistent agents, one shared host.*

*This is a working paper. Corrections welcome; §7 is opinion and §2 states the limits.*
