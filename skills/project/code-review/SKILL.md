---
name: code-review
description: "Use this skill when a diff needs reviewing against a fixed point — a branch, commit, or tag — before merge. Triggers: user says 'review this branch', 'code review', 'review since X', 'review the diff', or 'review this issue's implementation'. Runs two independent parallel reviews (Standards and Spec) and reports them side by side. Do NOT use this for the HITL/AFK routing decision made before implementation starts — that's preflight-check. This runs after implementation, before merge."
---

> **Jira MCP tool used by this skill:** `jira_get_issue` (Step 2)

# Code review — two-axis diff review

## Purpose

Review the diff since a fixed point along two independent axes:

- **Standards** — does the code conform to this repo's documented coding standards, plus a Fowler smell baseline?
- **Spec** — does the code faithfully implement the Jira issue's acceptance criteria and shape constraints?

Both axes run as parallel sub-agents so neither pollutes the other's context, then this skill aggregates their findings without merging or reranking them. A change can pass one axis and fail the other — reporting them separately stops one axis from masking the other.

## Inputs required

- A fixed point to diff against (branch, commit SHA, tag) — ask if not supplied
- The Jira issue key this diff implements (a `decompose-issues`-produced Story, or the PRD Epic if reviewing broader work) — ask if not supplied
- Git access to the working tree

## Process

### Step 1 — Pin the fixed point

Whatever the user supplied — a commit SHA, branch name, tag, `main`, `HEAD~5`. If they didn't specify one, ask for it.

Capture the diff command once: `git diff <fixed-point>...HEAD` (three-dot, so the comparison is against the merge-base). Also capture the commit list: `git log <fixed-point>..HEAD --oneline`.

Before going further, confirm the fixed point resolves (`git rev-parse <fixed-point>`) and the diff is non-empty. A bad ref or empty diff should fail here — not inside two parallel sub-agents.

### Step 2 — Identify the spec source

Ask the user for the Jira issue key this diff implements, if not already in context. Call `jira_get_issue` and extract:
- **Acceptance criteria** — the checklist from the issue body
- **Shape constraints** — the Change boundary / Required patterns / Prohibited patterns copied into the issue body from the PRD (see `decompose-issues` Step 6's issue template)

If the issue key resolves to a Story with a `## PRD reference` section but the shape constraints there feel thin, also fetch the parent PRD Epic via `jira_get_issue` for the full Shape Constraints section.

If no issue key is available and the user confirms there isn't one, skip the Spec sub-agent and note this in the final report.

### Step 3 — Identify the standards sources

Look for anything in the repo that documents how code should be written — a Java style guide, `CONTRIBUTING.md`, linter/checkstyle config with documented rationale. Most repos in this stack will have little or nothing here; that's expected, not an error.

On top of whatever the repo documents, the Standards axis always carries the **smell baseline** below — a fixed set of Fowler code smells (_Refactoring_, ch.3) that applies even when a repo documents nothing. Two rules bind it:

- **The repo overrides.** A documented repo standard always wins; where it endorses something the baseline would flag, suppress the smell.
- **Always a judgement call.** Each smell is a labelled heuristic ("possible Feature Envy"), never a hard violation — and, like any standard here, skip anything tooling already enforces.

Each smell reads *what it is* → *how to fix*; match it against the diff:

- **Mysterious Name** — a function, variable, or type whose name doesn't reveal what it does or holds. → rename it; if no honest name comes, the design's murky.
- **Duplicated Code** — the same logic shape appears in more than one hunk or file in the change. → extract the shared shape, call it from both.
- **Feature Envy** — a method that reaches into another object's data more than its own. → move the method onto the data it envies.
- **Data Clumps** — the same few fields or params keep travelling together (a type wanting to be born). → bundle them into one type, pass that.
- **Primitive Obsession** — a primitive or string standing in for a domain concept that deserves its own type. → give the concept its own small type.
- **Repeated Switches** — the same `switch`/`if`-cascade on the same type recurs across the change. → replace with polymorphism, or one map both sites share.
- **Shotgun Surgery** — one logical change forces scattered edits across many files in the diff. → gather what changes together into one module.
- **Divergent Change** — one file or module is edited for several unrelated reasons. → split so each module changes for one reason.
- **Speculative Generality** — abstraction, parameters, or hooks added for needs the spec doesn't have. → delete it; inline back until a real need shows.
- **Message Chains** — long `a.b().c().d()` navigation the caller shouldn't depend on. → hide the walk behind one method on the first object.
- **Middle Man** — a class or function that mostly just delegates onward. → cut it, call the real target direct.
- **Refused Bequest** — a subclass or implementer that ignores or overrides most of what it inherits. → drop the inheritance, use composition.

### Step 4 — Spawn both sub-agents in parallel

Send a single message with two `Agent` tool calls, both using the `general-purpose` subagent type.

**Standards sub-agent prompt** — include:
- The full diff command and commit list
- The list of standards-source files found in Step 3, **plus the smell baseline pasted in full** — the sub-agent has no other access to it
- The brief: "Report — per file/hunk where relevant — (a) every place the diff violates a documented standard: cite the standard (file + rule); and (b) any baseline smell you spot: name it and quote the hunk. Distinguish hard violations from judgement calls — documented-standard breaches can be hard, but baseline smells are always judgement calls, and a documented repo standard overrides the baseline. Skip anything tooling enforces. Under 400 words."

**Spec sub-agent prompt** — include:
- The diff command and commit list
- The fetched acceptance criteria and shape constraints from Step 2
- The brief: "Report: (a) acceptance criteria that are missing or partially implemented; (b) shape-constraint violations — anything touching a component outside the change boundary, or matching a listed prohibited pattern; (c) behaviour in the diff that wasn't asked for (scope creep); (d) criteria that look implemented but where the implementation looks wrong. Quote the issue text for each finding. Under 400 words."

If Step 2 found no issue key, skip the Spec sub-agent and note this in the final report instead of spawning it.

### Step 5 — Aggregate

Present the two reports under `## Standards` and `## Spec` headings, verbatim or lightly cleaned. Do not merge or rerank findings across axes — the separation is deliberate.

End with a one-line summary: total findings per axis, and the worst issue *within each axis* (if any). Don't pick a single winner across axes.

## Hard constraints

- Never merge or rerank Standards and Spec findings into one list. A change that passes Standards and fails Spec (or vice versa) must show as exactly that, not averaged into a single verdict.
- A documented repo standard always overrides the Fowler smell baseline where the two conflict.
- Never skip Step 1's ref/diff validation to save time — a bad ref inside a sub-agent produces a confusing partial report instead of a clean failure.
- Shape-constraint violations are Spec findings, not Standards findings, even though they read like conventions — they come from the PRD, not from the repo's general coding standards.
- Do not write review comments back to the Jira issue. This skill reports to the conversation only — posting review notes to Jira, if wanted, is a separate, explicit action by the user.
