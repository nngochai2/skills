---
name: decompose-issues
description: "Use this skill when a PRD and a Solution Detailed Design both exist and the work needs to be decomposed into Jira issues. Triggers: user says 'decompose issues', 'to-issues', 'create the issues', 'break this into tickets', or 'issue decomposition'. Do NOT use this skill before both the PRD and Solution Detailed Design are available — it decomposes from those two documents directly, not from test artifacts."
---

> **Jira MCP tools used by this skill:** `jira_create_issue`, `jira_link_issues`
> **Update tools:** `jira_update_issue`, `jira_transition_issue`, `jira_link_issues`

# Decompose issues — coupling-aware dependency DAG for Jira

## Purpose

Break a PRD + Solution Detailed Design into Jira issues sized by coupling and blast radius, ordered by dependency, and labelled for routing. The output is a dependency DAG (partially ordered, not a flat list) where blocking relationships are explicit and parallelism is determined by code graph analysis — not assumed.

This is the tracked milestone for measuring AI-assisted delivery in the project stream: the resulting issues are what the team lead/PM review against the PRD they came from. Treat the DAG and its issues as the reviewable artefact, not just a means to an end.

This is supervised AFK: the agent drafts, the tech lead validates the DAG before issues are created.

## Inputs required

- PRD (Jira issue key or content) — for shape constraints, behavior specification, and scope reference
- Solution Detailed Design — for impact list (components in scope)
- Code graph accessible via MCP (blast radius queries, component dependency lookup)
- **Ticket identifier** — ask the user at the start (see Step 0): the short slug used for `docs/<ticket>/` paths in this session
- **Jira project key** — ask the user at the start (see Step 0): "Which Jira project should I create issues in? (e.g. `PROJ`)"

If code graph is unavailable, proceed with Solution Detailed Design impact list only. Flag that blast radius is unconfirmed and all issues will be labelled `needs-preflight`.

## Process

### Step 0 — Establish the ticket identifier and Jira project

Ask the user before doing anything else:

1. "What ticket or identifier should I use for this session? (e.g. `epic-42`, `UC-014`, or a short slug — this becomes `docs/<ticket>/` in the repo)"
2. "Which Jira project should I create issues in? (e.g. `PROJ`) And what's the PRD's issue key, if it isn't already in context?"

Record both responses. Use `<ticket>` in all file path references and the Jira project key / PRD issue key in all MCP calls throughout this session.

### Step 1 — Build the slice inventory

Read the PRD's Behavior Specification and Scope sections alongside the Solution Detailed Design's impact list. Break the work into **vertical slices** (tracer bullets) — each slice is a thin but complete path through every layer the change touches (not a horizontal slice of one layer), independently demoable or verifiable.

For each slice:

```
Slice: <name>
Behavior covered: <which Behavior Specification condition(s) from the PRD this satisfies>
Components (from Solution Design impact list): <list>
```

Prefer many thin slices over few thick ones. This replaces scenario-based sizing — there is no Gherkin scenario to anchor to, so the slice boundary itself is the unit of work and deserves the same scrutiny a scenario boundary used to get.

### Step 2 — Query code graph for blast radius per slice

For each slice:

Query the code graph via MCP:
- Which Java classes/methods are touched by the components named in this slice?
- What is the dependency depth (how many layers out does a change propagate)?
- Does the blast radius overlap with any components that have active regulatory node links in the Document KG?

Record per slice:
```
Blast radius: [component list]
Dependency depth: N
Regulatory exposure: YES (node IDs) | NO
Overlap with other slices: [list of overlapping slice names]
```

### Step 3 — Determine granularity

For each slice, decide on issue granularity using this rubric:

**One slice → one issue when:**
- Blast radius is large (touches many components) — isolation keeps failures attributable
- Regulatory exposure is present — compliance changes deserve narrow, verifiable issues
- The slice has no overlap with other slices — no shared components

**Multiple slices → one issue when:**
- Slices share the same blast radius (same components in scope)
- Slices are logically sequential within one functional area
- Combined blast radius is small and well-understood
- All slices in the group share the same risk profile

Never group slices from different regulatory areas into one issue. Compliance failures must be traceable to a single, bounded change.

### Step 4 — Build the dependency DAG

For each issue, identify blockers:

An issue B is blocked by issue A when:
- A modifies a component that B reads or calls
- A resolves a shared data structure that B depends on
- A must be verified before B is logically reachable (e.g. B's slice extends behavior A introduces)

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
- Slice has a high-risk profile (per Solution Design)

**`AFK` (agent can proceed with Copilot + KG context):**
- Regulatory exposure: NO
- Dependency depth ≤ 3
- Blast radius is bounded and non-overlapping
- No high-risk profile

When in doubt, default to `HITL`. The cost of a wrong `AFK` label is higher than the cost of a wrong `HITL` label.

### Step 6 — Draft the issue body template

For each issue, generate this body:

```markdown
## Summary
<one sentence: what this issue implements and why>

## Acceptance criteria
- [ ] <criterion drawn from the PRD's Behavior Specification for this slice>
- [ ] <criterion drawn from the PRD's Behavior Specification for this slice>

## PRD reference
Section: <PRD section title and Jira issue key>
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
```

Verification of the acceptance criteria and closing the issue is a manual human call — there is no automated done-signal to check here (see Hard constraints).

### Step 7 — Present DAG for tech lead validation (before creating issues)

Present the full DAG to the tech lead:
- Parallel-start set (issues with no blockers)
- Dependency chains (longest path through the DAG)
- HITL vs AFK distribution
- Any issues where granularity decision was non-obvious (explain the reasoning)

Ask: "Does this DAG reflect how you'd sequence the work? Are there dependencies I've missed, or coupling risks the code graph didn't surface?"

Do not create Jira issues until the tech lead confirms the DAG.

### Step 8 — Create Jira issues via MCP

After tech lead confirmation, execute the following sequence using the Jira MCP tools.

**8a — Create all issues first (no links yet):**

For each issue in the DAG, call `jira_create_issue`:
```
title: "[<UC-ID>] <issue summary>"
description: <full body template from Step 6>
issue_type: "Story"
parent_key: <PRD epic issue key>
labels: ["implementation", "routing-hitl" or "routing-afk" (add "needs-preflight" if code graph was unavailable), "uc-<UC-ID>"]
```

Record each returned issue key — needed for Step 8b.

**8b — Wire the DAG dependencies:**

For every "A blocks B" edge in the DAG, call `jira_link_issues`:
```
issue_key: <key of A>
target_issue_key: <key of B>
link_type: "Blocks"
```

**8c — Confirm to tech lead:**

Present a summary table:

| Issue | Key | Routing | Blocks |
|-------|-----|---------|--------|
| [UC-ID] Title | PROJ-N | HITL/AFK | PROJ-M, PROJ-P |

State the parallel-start set (issues with no blockers, ready to pick up immediately).

## Updating issues after creation

Jira MCP has no comment/note tool — traceability for changes has to live in the issue description itself.

**Body or label corrections** — call `jira_update_issue` with the changed fields, then append a dated entry to the bottom of the description under a `## Change log` heading (create the heading if it doesn't exist yet):
```
## Change log
- <YYYY-MM-DD>: Changed <field>. Reason: <why it was wrong>. New value: <summary of new content>.
```
Never overwrite this section — append to it.

**Wrong dependency link** — the Jira MCP has no delete-link tool. Remove the incorrect link via the Jira UI, then call `jira_link_issues` to create the correct one.

**Issue is wrong or unnecessary** — the Jira MCP has no delete tool. Call `jira_transition_issue` to move it to whatever your board's "Won't Do" / "Cancelled" status is, and append a change-log entry explaining why. Do not attempt to work around this by inventing a delete mechanism.

**Complete redo of an issue** — call `jira_update_issue` to replace title and description entirely, then append a change-log entry explaining the redo reason.

## Hard constraints

- Never create issues before the tech lead validates the DAG. The DAG review is a gate, not a suggestion.
- Never group slices from different regulatory areas into one issue.
- Never label an issue `AFK` when regulatory exposure is YES. This overrides all other signals.
- Do not invent dependencies to enforce sequencing preferences. Dependencies must be traceable to code graph overlap or logical slice ordering.
- If the code graph is unavailable, every issue gets `needs-preflight` label — no issue gets `AFK` until a human runs the pre-flight check manually.
- Do not invent a "done signal" to replace the retired Cucumber-based one. Verifying acceptance criteria and closing the issue is a manual human judgment call, same as Pocock's original `to-issues` pattern — there is no automated substitute, and one should not be added without a concrete new verification mechanism to hook into.
- Always append to the description's `## Change log` section after any `jira_update_issue` call — never overwrite prior history.
