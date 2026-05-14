---
name: decompose-issues
description: "Use this skill when committed Gherkin scenarios and a Solution Detailed Design both exist and the work needs to be decomposed into GitLab issues. Triggers: user says 'decompose issues', 'to-issues', 'create the issues', 'break this into tickets', or 'issue decomposition'. Scenarios must be committed as failing tests before this skill runs. Do NOT decompose before Gherkin scenarios are committed — the scenarios are the unit of work."
---

> **GitLab MCP tools used by this skill:** `gitlab_create_issue`, `gitlab_link_issues`
> **Update tools:** `gitlab_update_issue`, `gitlab_create_issue_note`, `gitlab_delete_issue`, `gitlab_link_issues`

# Decompose issues — coupling-aware dependency DAG for GitLab

## Purpose

Break committed Gherkin scenarios into GitLab issues sized by coupling and blast radius, ordered by dependency, and labelled for routing. The output is a dependency DAG (partially ordered, not a flat list) where blocking relationships are explicit and parallelism is determined by code graph analysis — not assumed.

This is supervised AFK: the agent drafts, the tech lead validates the DAG before issues are created.

## Inputs required

- Committed .feature file(s) (scenarios as failing tests in the repo)
- PRD (GitLab issue URL or content) — for shape constraints and scope reference
- Solution Detailed Design — for impact list (components in scope)
- Code graph accessible via MCP (blast radius queries, component dependency lookup)
- **GitLab project path** — ask the user at the start: "Which GitLab project should I create issues in? (e.g. `group/project`)"

If code graph is unavailable, proceed with Solution Detailed Design impact list only. Flag that blast radius is unconfirmed and all issues will be labelled `needs-preflight`.

## Process

### Step 1 — Build the scenario inventory

List all committed scenarios grouped by .feature file. For each scenario:

```
Scenario: <name>
Feature file: <path>
Requirement ID: <from @requirement-id tag>
Regulatory area: <from @regulatory-area tag if present>
Risk tier: <from @risk-tier tag if present>
```

### Step 2 — Query code graph for blast radius per scenario group

For each scenario (or natural group of scenarios covering the same component area):

Query the code graph via MCP:
- Which Java classes/methods are touched by the components named in the Solution Detailed Design impact list for this scenario's scope?
- What is the dependency depth (how many layers out does a change propagate)?
- Does the blast radius overlap with any components that have active regulatory node links in the Document KG?

Record per scenario group:
```
Blast radius: [component list]
Dependency depth: N
Regulatory exposure: YES (node IDs) | NO
Overlap with other scenario groups: [list of overlapping scenario group names]
```

### Step 3 — Determine granularity

For each scenario or scenario group, decide on issue granularity using this rubric:

**One scenario → one issue when:**
- Blast radius is large (touches many components) — isolation keeps failures attributable
- Regulatory exposure is present — compliance changes deserve narrow, verifiable issues
- The scenario has no overlap with other scenarios — no shared components

**Multiple scenarios → one issue when:**
- Scenarios share the same blast radius (same components in scope)
- Scenarios are logically sequential within one functional slice
- Combined blast radius is small and well-understood
- All scenarios in the group share the same risk tier

Never group scenarios from different regulatory areas into one issue. Compliance failures must be traceable to a single, bounded change.

### Step 4 — Build the dependency DAG

For each issue, identify blockers:

An issue B is blocked by issue A when:
- A modifies a component that B reads or calls
- A resolves a shared data structure that B depends on
- A must pass its Cucumber scenario before B's scenario is logically reachable

Produce the DAG:
```
Issue: <title>
Depends on (blocked-by): [list of issue titles or NONE]
Blocks: [list of issue titles or NONE]
Can start immediately: YES | NO (waiting for: <blocker>)
```

Identify which issues have no blockers — these form the parallel-start set. In a coupled Java codebase, this set may be small. That is correct — do not artificially break dependencies to inflate parallelism.

### Step 5 — Assign routing labels

For each issue, assign labels based on the preflight gateway rules:

**`HITL` (human decision required):**
- Regulatory exposure: YES — hard rule, no exceptions
- Dependency depth > 3 (high blast radius complexity)
- Blast radius overlaps with more than two other issues (coupling risk)
- Scenario has a `@risk-tier::high` tag

**`AFK` (agent can proceed with Copilot + KG context):**
- Regulatory exposure: NO
- Dependency depth ≤ 3
- Blast radius is bounded and non-overlapping
- No `@risk-tier::high` tag

When in doubt, default to `HITL`. The cost of a wrong `AFK` label is higher than the cost of a wrong `HITL` label.

### Step 6 — Draft the issue body template

For each issue, generate this body:

```markdown
## Summary
<one sentence: what this issue implements and why>

## Scenarios owned
- [ ] `<scenario name>` in `<feature-file-path>`
- [ ] `<scenario name>` in `<feature-file-path>`

## PRD reference
Section: <PRD section title and GitLab issue link>
Shape constraints: <copy relevant shape constraints section from PRD>

## Blast radius (from code graph pre-flight)
Components in scope: [list]
Dependency depth: N
Regulatory exposure: YES (KG node IDs) | NO

## Routing
Label: HITL | AFK
Reason: <one sentence justifying the label>

## Blocked by
- <issue title> | NONE

## Blocks
- <issue title> | NONE

## Done signal
All owned scenarios pass in the test environment. No regressions in the full Cucumber suite.
```

### Step 7 — Present DAG for tech lead validation (before creating issues)

Present the full DAG to the tech lead:
- Parallel-start set (issues with no blockers)
- Dependency chains (longest path through the DAG)
- HITL vs AFK distribution
- Any issues where granularity decision was non-obvious (explain the reasoning)

Ask: "Does this DAG reflect how you'd sequence the work? Are there dependencies I've missed, or coupling risks the code graph didn't surface?"

Do not create GitLab issues until the tech lead confirms the DAG.

### Step 8 — Create GitLab issues via MCP

After tech lead confirmation, execute the following sequence using the GitLab MCP tools.

**8a — Create all issues first (no links yet):**

For each issue in the DAG, call `gitlab_create_issue`:
```
project_id: <project path confirmed at session start>
title: "[<UC-ID>] <issue summary>"
description: <full body template from Step 6>
labels: "type::implementation,routing::HITL,uc::<UC-ID>"
         or "type::implementation,routing::AFK,uc::<UC-ID>"
         or "type::implementation,routing::AFK,needs-preflight,uc::<UC-ID>"
```

Record each returned `iid` and `web_url` — these are needed for Step 8b.

**8b — Wire the DAG dependencies:**

For every "A blocks B" edge in the DAG, call `gitlab_link_issues`:
```
project_id: <project path>
issue_iid: <iid of A>
target_project_id: <project path>
target_issue_iid: <iid of B>
link_type: "blocks"
```

**8c — Confirm to tech lead:**

Present a summary table:

| Issue | IID | URL | Routing | Blocks |
|-------|-----|-----|---------|--------|
| [UC-ID] Title | #N | url | HITL/AFK | #M, #P |

State the parallel-start set (issues with no blockers, ready to pick up immediately).

## Updating issues after creation

When the user flags an issue as not qualified, use the following tools in order:

**Body or label corrections** — call `gitlab_update_issue` with the changed fields, then immediately call `gitlab_create_issue_note` with a note explaining what changed and why:
```
body: "**Updated by decompose-issues review**\n\nChanged: <field>\nReason: <why it was wrong>\nNew value: <summary of new content>"
```

**Wrong dependency link** — GitLab does not support editing links. Delete the incorrect link via the GitLab UI (or API), then call `gitlab_link_issues` to create the correct one.

**Issue is wrong or unnecessary** — call `gitlab_delete_issue` if the issue should not exist at all. Prefer `gitlab_close_issue` if the issue should be kept for audit history but not actioned.

**Complete redo of an issue** — call `gitlab_update_issue` to replace title and description entirely, then `gitlab_create_issue_note` explaining the redo reason. Do not delete and recreate unless the IID needs to be freed.

## Hard constraints

- Never create issues before the tech lead validates the DAG. The DAG review is a gate, not a suggestion.
- Never group scenarios from different regulatory areas into one issue.
- Never label an issue `AFK` when regulatory exposure is YES. This overrides all other signals.
- Do not invent dependencies to enforce sequencing preferences. Dependencies must be traceable to code graph overlap or logical scenario ordering.
- If the code graph is unavailable, every issue gets `needs-preflight` label — no issue gets `AFK` until a human runs the pre-flight check manually.
- The done signal is Cucumber scenario pass, not code merge. Make this explicit in every issue body.
- Always call `gitlab_create_issue_note` after any `gitlab_update_issue` call — every change must be traceable.
