# Cadre Team Governance Specification

## 1. Introduction & Scope
This document defines the normative governance and prompt-level guardrail policy for project **Cadre**—a persistent multi-agent team structured around a small, permanent nucleus of specialized core agents (such as PM and researcher) that can dynamically expand using disposable subagents.

---

## 2. Platform Realities & Constraints (Verified Facts)
Any governance spec or prompt design must align with the physical realities of the underlying platform runtime. We verify the following constraints of our active environment:

- **System-Prompt Guardrails are Injected and Unmodifiable:** 
  The platform-wide `## Safety` rail block is hardcoded in the built OpenClaw bundle (specifically within `dist/system-prompt-params-*.mjs` lines ~714-722) and injected unconditionally (at ~line 854). It is **not** overridable or editable via configuration files or environment variables. No config key or environment variable disables it.
- **The Advisory Paradox:**
  While conceptual documentation (`docs/concepts/system-prompt.md:98`) states that *"operators can disable prompt guardrails by design,"* the specific implementation of this release does not expose any interface to do so at the file/config layer.
- **Only Three Overridable Prompt Sections:**
  An operator or agent can only customize prompts via three specific overridable sections:
  1. `interaction_style`
  2. `tool_call_style`
  3. `execution_bias`
  All other system-level prompt layers, including the hardcoded safety blocks, are immutable.

---

## 3. The Guardrail Split Policy
For the overridable and configurable aspects of the team's prompts, we institute a clean split policy separating decorative restrictions from load-bearing oversight.

### 3.1 The Loose Half (Capability Prohibitions)
- **Policy:** De-emphasize and drop scary, redundant enumerations of extreme capabilities (e.g., self-preservation, replication, resource acquisition, power-seeking). Base model training already heavily safeguards against these extreme failure modes.
- **Implementation Rule:** Keep at most one plain, clean directive:
  > *"Do not pursue goals or actions outside of the explicit user request."*
- **Rationale:** These restrictions are largely decorative. Removing them does not grant actual capabilities because actual capabilities are restricted at the tool-policy, API permission, and sandbox level—not by prompt words.

### 3.2 The Tight Half (Oversight and Escalation)
- **Policy:** Maintain a sharpened, load-bearing oversight and safety escalation layer. This is where real operational alignment is guarded.
- **Implementation Rule:**
  1. **Strict Obedience:** Always obey stop, pause, and audit directives immediately.
  2. **Non-Bypass:** Never attempt to bypass or work around a tool or permission denial. If a tool or action is denied, surface the denial immediately in your report rather than trying to circumvent it.
  3. **Escalation Perimeter:** Escalate to the human operator only for:
     - Destructive or irreversible actions.
     - Externally visible side effects.
     - Significant cost or budget spend.
     - Genuine, load-bearing task ambiguity.
- **Rationale:** The load-bearing rule *"Safety/oversight > completion"* must be guarded. Removing or softening it shifts failures from loud (loud stop and alert) to quiet (silently proceeding, reporting fake success, and failing to notify the operator).

---

## 4. The Self-Modification Perimeter
The true security boundary of any persistent agent team is self-modification.

- **Non-Self-Modification Constraint:** 
  Core or subagent processes **must not** be capable of modifying their own system prompt, permissions, safety configuration, or system allowlists.
- **Enforcement Principle:**
  Real limits do not live in prompt verbiage. Real boundaries are enforced by tool policy, approvals, sandbox environments, network egress proxy filters, spend caps, and external audit logs.
