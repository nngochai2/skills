---
name: grill-me
description: "Use this skill when a Use Case description document exists and the team needs to interrogate it before writing a PRD. Triggers: user shares a UC document (via KG or direct input) and says 'grill me', 'stress-test this UC', 'start the grill-me session', or 'what are we missing'. This skill conducts a structured interrogation that surfaces regulatory gaps, unresolved decisions, and KG coverage gaps — and produces the decisions checklist that gates PRD writing. Do NOT use during PRD writing, solution design, or implementation."
---

# Grill me — UC interrogation with KG context

## Purpose

Interrogate a Use Case document relentlessly until every decision is resolved and every regulatory gap is surfaced. No PRD or code is written during this session. The output is a decisions checklist that gates PRD entry, plus KG gap note drafts categorised by which downstream phase they block.

The tech lead is in the hot seat. The KG sharpens the questions — it does not replace the interrogation.

## Inputs required before starting

- Use Case document (uploaded, pasted, or given from knowledge graph)
- Document KG accessible via MCP (query regulatory nodes, existing scenarios, incident post-mortems)
- Code graph via MCP if available (structural reach — which components does this UC touch)

If the Code graph is unavailable, proceed with Document KG only.
- If relevant architecture notes found in the documetn KG, fetch them then double check in the codebase using semantic search if available, esle use traditional search
- Else, flag that blast radius cannot be confirmed until Solution Detailed Design.

## Process

### Step 1 — KG orientation (do before asking any questions)

Query the Document KG via MCP for:
- Regulatory nodes related to the UC domain (e.g. tax rules, invoice format rules, e-reporting mandates)
- Existing Gherkin scenarios from related functional areas (for consistency and gap detection)
- Incident post-mortems linked to components the UC likely touches

If Code graph is available, query for structural reach: what components are likely in scope based on the UC description.

Build a private mental map of: known regulatory coverage, known gaps, known failure modes in the area.

### Step 2 — Identify the decision tree

Before asking any question, enumerate silently the decisions that must be resolved:
- Regulatory: which rules apply, how are edge cases handled, what is out-of-scope by regulation
- Behavioral: what are the exact pre/postconditions, what happens on error paths
- Architectural: which components are touched, what is the intended change boundary (shape)
- Data: what are the data contracts, what validation is required

### Step 3 — Interrogate systematically

Ask one question at a time. Do not ask multiple questions in one message.

Work through the decision tree branch by branch. For each branch:
- Ask the question
- If the answer resolves the decision: record it and move to the next branch
- If the answer is vague or deferred: push back. "That's not a decision — what specifically will happen when X?" Do not move on until the decision is resolved.
- If the answer reveals a new branch: explore it before moving on.

Use KG knowledge to ask sharper questions. If the KG has a regulatory node that applies, reference it: "The KG shows this flow is governed by Rule Y. Does this UC need to handle the exception case defined in that rule?"

### Step 4 — Surface KG gaps

For every piece of regulatory or domain knowledge that was needed during interrogation but was NOT in the KG, record it as a gap note draft.

Categorise each gap by which phase it blocks:

| Tier | Condition | Blocks |
|------|-----------|--------|
| 1 — Regulatory | Missing rule that directly governs this UC | PRD writing (hard block) |
| 2 — Coverage | Missing incident post-mortem or related scenario | Gherkin generation |
| 3 — Structural | Missing architectural convention or pattern | Issue decomposition |
| 4 — Async | Peripheral context, nice-to-have | Nothing immediately |

Tier 1 gaps must be resolved before this session ends or explicitly accepted as a known risk with tech lead sign-off. Never silently proceed past a Tier 1 gap.

### Step 5 — Produce outputs

**Decisions checklist** (gate to PRD): a numbered list of every resolved decision. Format:
```
1. [RESOLVED] Regulatory scope: this UC applies Rule Y in full. Exception case Z is out of scope.
2. [RESOLVED] Error path: on validation failure, return error code E with message M. Do not persist.
3. [DEFERRED — TECH LEAD ACCEPTED RISK] Blast radius: code graph unavailable; assumed bounded to Component X.
```

**KG gap note drafts** (one per gap, in Obsidian format):
```
---
tags: [kg-gap, tier-1, regulatory, uc-reference/<UC-ID>]
status: pending
blocks: prd
---

# Gap: <short description>

## What is missing
<what knowledge is absent>

## Why it matters
<what breaks if this stays empty>

## Suggested source
<where to find this — regulation document, SME, existing code>
```

## Hard constraints

- Never produce a PRD during this session. If the user asks, say the session must complete first.
- Never mark a decision as resolved unless a concrete, specific answer was given. "We'll handle it later" is not an answer.
- Never skip a Tier 1 gap. Surface it, push back, get explicit tech lead sign-off if unresolvable now.
- Never infer regulatory intent from the code. Code describes current behaviour, not regulatory requirement.
- Do not use answers from the code graph to fill in regulatory decisions — code may already be wrong.
