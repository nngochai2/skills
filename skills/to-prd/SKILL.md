---
name: to-prd
description: "Use this skill when a Grill-me session is complete and a decisions checklist exists. Triggers: user says 'write the PRD', 'to-prd', 'produce the PRD', or 'turn this into a PRD'. The decisions checklist from Grill-me must be present in context or explicitly provided. Do NOT use without a completed decisions checklist — if one is missing, run the grill-me skill first."
---

> **GitLab MCP tool used by this skill:** `gitlab_create_issue` (Step 5)

# To PRD — compliance-aware product requirements document

## Purpose

Synthesise the decisions checklist from Grill-me into a structured PRD. The PRD is the single source of truth for downstream steps: Gherkin generation uses it, code review uses it as anchor, and agents read it for shape constraints during implementation.

The PRD must carry shape constraints written by the Developer. This is non-negotiable.

## Inputs required

- Decisions checklist from completed Grill-me session
- Developer input on shape constraints (see Step 2 — ask explicitly if not provided)
- KG node references for all regulatory claims (available from Grill-me KG orientation)

## Process

### Step 1 — Check prerequisites

Verify the decisions checklist is complete: no unresolved decisions, no unacknowledged Tier 1 gaps. If any exist, stop and tell the user which items must be resolved first.

### Step 2 — Elicit shape constraints (mandatory)

Before writing any PRD content, ask the Developer:

> "Before I write the PRD, I need the shape constraints for this change. Please describe:
> 1. Which components are the intended change boundary — what should and should not be modified?
> 2. Which patterns should the implementation follow (e.g. Java service layer, not stored procedure; existing error handling convention X)?
> 3. Which anti-patterns are explicitly ruled out for this UC?
>
> These constraints will be written into the PRD so that agents during implementation and decomposition are bound by them."

Do not proceed without concrete answers. If the Developer says "I don't know yet", reply: "Shape constraints that are unknown at PRD time become implementation decisions made by the agent or Copilot — often incorrectly. Please make them explicit now, even if approximate."

### Step 3 — Write the PRD

Structure:

```markdown
# PRD: <UC Title>

## Overview
One paragraph. What this UC does, why it exists, what regulatory context governs it.
KG references: [list of KG node IDs from the Grill-me session]

## Behavior specification
Precise description of system behavior. Written as conditions, not steps.
For each behavior:
- Pre-condition
- Trigger
- Expected outcome (success path)
- Expected outcome (each error path resolved in Grill-me)

## Scope
What this UC changes. Components, data contracts, API surfaces.

## Out of scope
Explicit exclusions agreed in Grill-me. Reference the decision checklist item for each.

## Shape constraints
[MANDATORY — written by Developer, not inferred]

### Change boundary
Which components may be modified. Which may not.

### Required patterns
Implementation patterns this change must follow.
Example: "Validation logic goes in the Java service layer. No new PL/SQL procedures."

### Prohibited patterns
Anti-patterns explicitly ruled out.
Example: "Do not extend Oracle View V_INVOICE_LINES. Route through InvoiceService instead."

## Acceptance criteria
Reference to Gherkin scenarios (to be generated in draft-gherkin step).
Format: "Scenarios in <feature-file-name>.feature — generated from this PRD."

## Known gaps and deferred decisions
List any Grill-me gaps that are Tier 2–4 (not blocking PRD) with their tier and the phase they must be resolved before.
```

### Step 4 — Validate before publishing

Check:
- Shape Constraints section contains concrete content (not placeholder text)
- Every claim in Behavior Specification is traceable to a KG node or a resolved Grill-me decision
- Out of Scope explicitly references at least one decision checklist item
- No contradictions between Behavior Specification and KG regulatory nodes

If any check fails, fix before publishing.

### Step 5 — Publish to GitLab via MCP

Ask the user: "Which GitLab project should the PRD issue be created in? (e.g. `group/project`)"

Then call `gitlab_create_issue`:
```
project_id: <confirmed project path>
title: "PRD: <UC Title>"
description: <full PRD markdown from Step 3>
labels: "type::prd"
```

Confirm the returned `web_url` to the user.

## Hard constraints

- Shape Constraints section cannot be empty, cannot say "TBD", and cannot be written by the AI without Developer input. If the Developer refuses to provide shape constraints, note this explicitly in the PRD under a "Shape constraints deferred" heading and flag it as a risk.
- Every regulatory claim must reference a KG node ID. Do not assert regulatory requirements from training knowledge — only from the KG.
- Do not synthesise scope from code graph current state. Scope comes from decisions, not from what the code currently does.
- The PRD is the anchor for code review. Write it with that consumer in mind: reviewers will ask "does this implementation match the PRD?" — it must be unambiguous enough to answer that question.
