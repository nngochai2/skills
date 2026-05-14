---
name: preflight-check
description: "Use this skill when a GitLab issue is about to enter implementation and needs a code graph pre-flight check. Triggers: user says 'preflight', 'pre-flight check', 'check this issue before we start', or when an issue moves from Kanban 'Ready' to 'In Progress'. The issue body (with blast radius and routing label) must be present. Do NOT skip this step even for AFK-labelled issues — routing labels from decomposition are based on the state at decomposition time, not current state."
---

> **GitLab MCP tools used by this skill:** `gitlab_get_issue` (read issue body), `gitlab_update_issue` (relabel if routing changes), `gitlab_create_issue_note` (log the preflight result)

# Preflight check — code graph pre-flight before implementation

## Purpose

Confirm that the code graph's current state matches the assumptions made during issue decomposition. Catch drift — changes made since decomposition that affect the blast radius or introduce new coupling — before an agent or developer starts implementation work.

Apply the gateway rules and produce a final routing decision with rationale. Update the issue label if the routing has changed.

## Inputs required

- GitLab issue body (blast radius list, routing label, scenarios owned)
- Code graph accessible via MCP (current state query)
- Solution Detailed Design (for shape constraints reference)
- Document KG accessible via MCP (regulatory node lookup)

## Process

### Step 1 — Extract the decomposition-time blast radius

From the issue body, extract:
- Components in scope (blast radius list from decomposition)
- Dependency depth recorded at decomposition
- Regulatory exposure recorded at decomposition (YES/NO + KG node IDs)
- Current routing label (HITL/AFK)

This is the baseline. Everything in Steps 2–4 is measured against it.

### Step 2 — Query code graph for current state

For each component in the blast radius, query the code graph via MCP:

- Has the component changed since the last code graph build? (compare against known state at decomposition time if available)
- Have any new inbound or outbound dependencies been introduced?
- Has the component's dependency depth changed?
- Are there any components now in the transitive blast radius that were not present at decomposition time?

Record as a diff:
```
Component: <name>
Status: UNCHANGED | CHANGED | NEW DEPENDENCY ADDED | REMOVED
Detail: <what specifically changed, if anything>
```

### Step 3 — Re-check regulatory exposure

Even if the blast radius appears unchanged, query the Document KG for all components now in scope:

- Do any components in the current blast radius have regulatory node links that were NOT present at decomposition time?

This matters because KG enrichment is ongoing — a node may have been added to the KG between decomposition and implementation that changes the regulatory exposure of this issue.

Record:
```
Regulatory exposure at decomposition: YES (nodes) | NO
Regulatory exposure now: YES (nodes) | NO
Delta: NONE | NEW EXPOSURE (list new nodes) | EXPOSURE REMOVED (list removed nodes)
```

### Step 4 — Apply gateway rules

**Rule 1 — Regulatory exposure (hard rule, no exceptions):**
If regulatory exposure is YES (either at decomposition or now), the routing is HITL.
This rule cannot be overridden by any other signal.

**Rule 2 — Drift detected:**
If any component in the blast radius has changed since decomposition, escalate to HITL regardless of the decomposition-time label.
Rationale: the agent's implementation plan was based on the state at decomposition. If the code has changed, that plan may no longer be valid.

**Rule 3 — Dependency depth (secondary):**
If current dependency depth > 3, routing is HITL.

**Rule 4 — Blast radius expansion:**
If new components are now in the transitive blast radius that were not at decomposition time, escalate to HITL.

**Rule 5 — Clean bill of health:**
If Rules 1–4 all clear: routing is AFK. Proceed with Copilot + KG context.

### Step 5 — Produce routing decision

```
PREFLIGHT RESULT: <issue title>

Code graph state: CURRENT | DRIFTED (detail)
Regulatory exposure: YES (nodes) | NO
Dependency depth: N
Blast radius delta: NONE | EXPANDED (list new components)

Routing decision: HITL | AFK
Governing rule: <Rule 1–5 that determined the outcome>
Reason: <one sentence>

Previous label: HITL | AFK
Label update required: YES | NO
```

If the label changes (AFK → HITL or HITL → AFK), call `gitlab_update_issue` with the corrected `labels` value, then call `gitlab_create_issue_note` with the full preflight result block as the note body so the routing change is traceable.

### Step 6 — Scope Copilot context (AFK issues only)

If routing is AFK, produce an issue-scoped context block for Copilot:

```markdown
## Copilot context for issue: <title>

### Relevant KG nodes
<list of KG node IDs and descriptions relevant to this issue's scope>
Query: [Cypher query used to retrieve these nodes]

### Shape constraints (from PRD)
<copy of shape constraints section from PRD>

### Components in scope
<blast radius list — only these components should be modified>

### Components explicitly out of scope
<list components adjacent to the blast radius that must NOT be modified>

### Relevant existing scenarios (for pattern consistency)
<list of KG scenario references from related areas>
```

This context block goes into the issue body or is provided directly to Copilot as a system prompt supplement.

### Step 7 — Brief the implementer

For HITL issues: summarise what the human decision-maker needs to know — which regulatory nodes are in play, what the blast radius looks like, what the shape constraints say. Do not make the implementation decision. Present the evidence and wait.

For AFK issues: confirm Copilot context is scoped and the issue is ready for agent pickup.

## Hard constraints

- Never skip this step even if the issue was labelled AFK at decomposition. The pre-flight always runs.
- Never override Rule 1 (regulatory exposure). If a regulatory node is in the blast radius, routing is HITL. No complexity argument, deadline pressure, or "it's a small change" justification overrides this.
- Never produce Copilot context for a HITL issue. The human decision happens before implementation context is set.
- If the code graph is unavailable (build stale, jQAssistant not run), escalate to HITL for all issues. Do not proceed with AFK routing on an unverified blast radius.
- Drift is not a blocker — it is a routing signal. A drifted blast radius means HITL, not "stop work". The human decides whether to proceed, adjust scope, or re-decompose.
