# AEGIS Trust Tier Specification

> Version: 1.0 (draft)
> Status: Public — open for community review

---

## Overview

The AEGIS Trust Tier Model is a declarative action classification framework for agentic systems. It provides a principled, auditable answer to the question every autonomous agent eventually faces: **should I act, notify, or stop and ask?**

The model classifies every action an agent can take into one of three tiers based on two axes: **reversibility** and **blast radius** (the scope of systems or people affected if the action goes wrong).

---

## The Three Tiers

### T0 — Silent
**Agent acts autonomously. No notification required.**

T0 actions are fully reversible, local in scope, and affect no shared systems or external parties. The cost of acting without asking is negligible; the cost of requiring approval for every T0 action would make the agent unusable.

**Examples:**
- Reading files or querying a local database
- Writing to a local workspace or temp directory
- Running analysis, summarization, or classification
- Searching memory or knowledge graph

**Requirements:** None. Agent proceeds immediately.

---

### T1 — Notify
**Agent acts, then notifies the operator in real time.**

T1 actions have meaningful side effects — they may be harder to reverse, affect shared state, or be visible to other systems — but the risk profile does not justify blocking execution for approval. The operator is informed so they can intervene if needed.

**Examples:**
- Writing to a shared database or configuration file
- Creating a task, mandate, or scheduled job
- Posting to an internal message queue or notification channel
- Committing code to a non-main branch

**Requirements:** Audit log entry written before execution. Operator notification delivered within the action's execution window.

---

### T2 — Approval
**Agent pauses. Operator must confirm before execution proceeds.**

T2 actions are hard to reverse, externally visible, affect shared infrastructure, or carry legal/financial consequences. The 60-second enforced delay between confirmation and execution is intentional — it creates a window for the operator to cancel even after approving, and it prevents automated approval chains from bypassing the intent of the tier.

**Examples:**
- Pushing code to a main/production branch
- Sending external communications (email, Slack, client-facing messages)
- Modifying CI/CD pipelines or production infrastructure
- Deleting files, databases, branches, or containers
- Any action that involves external parties or shared systems beyond the local environment

**Requirements:**
1. Dual confirmation (operator approval + system acknowledgment)
2. 60-second enforced delay after confirmation, before execution
3. Audit log entry written at confirmation time and at execution time
4. Action must be cancellable during the delay window

---

## Classification Decision Matrix

When classifying a new action type, apply these questions in order:

| Question | Yes → | No → |
|----------|-------|-------|
| Is this action fully reversible? | Continue | T2 |
| Does this affect only the local environment? | Continue | T1 or T2 |
| Is this visible to external parties or shared systems? | T2 | Continue |
| Could this affect a billing, legal, or compliance boundary? | T2 | Continue |
| Is operator awareness useful even if not required? | T1 | T0 |

---

## Action Metadata Schema

Every action in an AEGIS-compatible system should carry the following metadata fields:

```json
{
  "action_id": "uuid",
  "action_type": "string",
  "trust_tier": "T0 | T1 | T2",
  "description": "Human-readable description of what this action does",
  "blast_radius": "local | internal | external",
  "reversible": true,
  "injection_scan_score": 0.0,
  "requires_confirmation": false,
  "audit_log_required": false
}
```

| Field | Description |
|-------|-------------|
| `trust_tier` | T0, T1, or T2 |
| `blast_radius` | `local` (no shared systems), `internal` (shared but contained), `external` (visible outside the platform) |
| `reversible` | Whether the action can be undone without data loss |
| `injection_scan_score` | 0.0–1.0 score from pre-execution injection analysis; T2 actions with score > 0.7 should be rejected, not just flagged |
| `requires_confirmation` | True for all T2 actions |
| `audit_log_required` | True for all T1 and T2 actions |

---

## Audit Log Requirements

All T1 and T2 actions must produce an immutable audit record containing:
- `action_id` — unique identifier for the action
- `agent_id` — the agent or process that initiated the action
- `tier` — T1 or T2
- `description` — what the action did
- `outcome` — `completed`, `cancelled`, `failed`, or `escalated`
- `confirmed_by` — operator identifier (T2 only)
- `confirmed_at` — ISO 8601 timestamp of operator confirmation (T2 only)
- `executed_at` — ISO 8601 timestamp of actual execution
- `delay_enforced_seconds` — enforced delay between confirmation and execution (T2 only)

---

## Design Rationale

The trust tier model exists because the alternative — hard-coding per-action rules or relying on the agent's own judgment about when to ask — produces brittle systems. Hard-coded rules don't generalize; agent self-reporting is not an audit trail.

The 60-second T2 delay is not a UX choice. It is a safety mechanism. Confirmation and execution are intentionally separated so that a fast-moving approval chain (human clicks approve without reading) still produces a window for cancellation. In practice, most T2 cancellations happen in this window.

The injection scan score requirement for T2 actions addresses prompt injection at the action boundary — the point where agent reasoning translates into system effects. An agent that has been manipulated into requesting a T2 action via injected instructions should fail the scan before the operator ever sees the confirmation dialog.

---

## Contributing

This specification is maintained by [Alva Systems Architecture LLC](https://alvasystemsarchitecture.com). 

We welcome discussion on edge cases, classification questions, and extensions to the schema. Open an issue or submit a PR.

---

*AEGIS Trust Tier Specification v1.0 (draft)*
*License: Apache 2.0*
