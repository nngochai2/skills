---
name: annotate-kg
description: "Use this skill when a Human decision has been made during implementation (HITL routing) and the rationale needs to be captured in the Document KG. Triggers: user says 'annotate KG', 'log this decision', 'document why', 'add this to the KG', or after a HITL issue resolution is confirmed. Do NOT use for AFK issues — AFK implementation decisions are routine and do not require KG annotation unless they reveal something unexpected."
disable-model-invocation: true
---

# Annotate KG — capture human decisions as KG artefacts

## Purpose

Human decisions made during HITL implementation contain institutional knowledge that is not in the code, not in the PRD, and will not survive the next developer rotation. Capturing them in the Document KG closes the enrichment loop: the next Grill-me session will be sharper because this decision exists as a node.

A decision not captured here will be rediscovered, often incorrectly, by a future developer or agent.

## Inputs required

- Description of the human decision made (what was decided)
- The HITL issue reference (Jira issue key)
- The regulatory context (which KG nodes were in play during pre-flight)
- The alternative(s) considered and why they were rejected

If the user cannot articulate the alternative(s) considered, ask before producing the note. A decision recorded without its rejected alternatives is half a decision — future agents will not know what was ruled out.

## Process

### Step 0 — Establish the ticket identifier

If a ticket identifier is not already established in the conversation context, ask the user before doing anything else:

> "What ticket or identifier should I use for this session's output folder? (e.g. `epic-42`, `UC-014`, or a short slug — this becomes `docs/<ticket>/` in the repo)"

Record the response as `<ticket>` and use it consistently for this session.

### Step 1 — Elicit the full decision record

Ask the following if not already provided:

1. "What was decided? State it as a rule or constraint, not as a narrative." (e.g. "VAT recalculation must always be triggered by InvoiceService.recompute(), never directly by the Oracle view" — not "we decided to use the service layer")

2. "What alternatives were considered?" List them.

3. "Why was each alternative rejected?" One sentence per alternative.

4. "Is this decision time-bounded? Does it apply until a specific condition changes (e.g. a regulation is updated, a component is refactored)?" If yes, record the condition.

5. "Which KG regulatory nodes does this decision relate to?" Retrieve node IDs from pre-flight output.

6. "Should this decision constrain future Grill-me sessions for related UCs?" If yes, it becomes a Tier 1 node — the next grill-me in this area will surface it as a hard constraint.

### Step 2 — Draft the Obsidian note

```markdown
---
tags: [decision, hitl, uc-reference/<UC-ID>, <regulatory-area-tag>]
status: active
kg-tier: 1
related-nodes: [<KG node ID list>]
issue-reference: <Jira issue key>
date: <YYYY-MM-DD>
expires-when: <condition or "indefinite">
---

# Decision: <imperative statement of the decision — one sentence>

## Context
<Two to three sentences: what situation triggered this decision, what was at stake>

## Decision
<The decision stated as a durable rule or constraint.>

Example format: "When X occurs in context Y, the system must use approach Z. Direct use of alternative A is prohibited."

## Alternatives considered

### Alternative 1: <name>
**Rejected because:** <one sentence>

### Alternative 2: <name>
**Rejected because:** <one sentence>

## Regulatory basis
KG nodes in scope at the time of this decision:
- [[<node-ID>]] — <node description>
- [[<node-ID>]] — <node description>

## Applicability
Future UCs touching <component list> should consult this decision during Grill-me.
This decision applies until: <condition or "indefinite">.

## Issue reference
[[jira-issue/<issue-key>]] — <issue title>
```

### Step 3 — Classify the node tier

Determine how this decision node should be treated in future Grill-me sessions:

**Tier 1 — Hard constraint (surface as a blocking question):**
- Decision involves a regulatory rule directly
- Decision establishes a component boundary that must not be crossed
- Getting this wrong in a future UC would produce a compliance failure

**Tier 2 — Coverage note (surface as a coverage check):**
- Decision establishes a pattern preference, not a hard boundary
- Getting this wrong in a future UC would produce technical debt, not a compliance failure

**Tier 3 — Incident pattern (surface as a risk reminder):**
- Decision was made to avoid a known failure mode
- Future UCs in the same area should be reminded of the incident

Record the tier in the `kg-tier` frontmatter field. The Grill-me skill uses this to determine how forcefully to surface the node during interrogation.

### Step 4 — Identify wikilinks

The Obsidian pipeline uses wikilinks for relationship extraction. Before committing:

- Link to related regulatory nodes using `[[node-ID]]` syntax
- Link to the Jira issue using `[[jira-issue/<issue-key>]]`
- Link to any related existing decision notes using `[[decision/<title>]]`
- If this decision supersedes a previous decision, link to it and mark the previous note as `status: superseded`

### Step 5 — Confirm and hand off for commit

Present the draft note to the user. Ask:

> "Does this accurately capture the decision and its rationale? Once committed to Obsidian and ingested by the KG pipeline, this will appear in future Grill-me sessions for related areas."

After confirmation: the note is ready for commit to the Obsidian vault. Remind the user to run the KG pipeline (`build_knowledge_graph.py`) after commit so the node is ingested into Neo4j before the next Grill-me session.

## Hard constraints

- Never draft a decision note without the rejected alternatives. A decision without ruled-out alternatives is not a decision — it is a description of what happened. Future agents need to know what not to do as much as what to do.
- Never use narrative prose for the Decision section. State it as a rule: "When X, use Y. Do not use Z." Narrative descriptions erode over time and are harder for agents to reason against.
- Never mark a decision as `kg-tier: 1` without confirming with the tech lead. Tier 1 nodes block PRD writing in future sessions — they carry real workflow cost.
- If the decision reveals a gap in the Document KG that was not caught at Grill-me time, produce a Tier 1 or Tier 2 KG gap note alongside the decision note.
- Do not commit on behalf of the user. The note is a draft until the user confirms and commits to Obsidian manually. KG quality depends on deliberate curation, not automated ingestion.
