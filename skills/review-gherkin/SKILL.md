---
name: review-gherkin
description: "Use this skill when LLM-generated Gherkin scenario drafts exist and a tester needs to review them. Triggers: user says 'review gherkin', 'tester review', 'check the scenarios', or 'review the drafts'. The draft .feature file(s) and coverage report from draft-gherkin must be present. Do NOT use this skill to generate scenarios — use draft-gherkin for that."
---

# Review Gherkin — tester review, judgment, and routing

## Purpose

The tester is the domain expert and quality gate — not the author. This skill structures the review so the tester's genuine contributions are captured efficiently:
- System knowledge that the LLM cannot have
- Granularity judgment (too broad, too narrow, duplicate)
- Ambiguity routing (back to PRD or Solution Design — not the same thing)
- Risk weighting (regulatory areas needing more coverage than the LLM generated)

Reviewing a draft is faster than writing from scratch. The LLM handles mechanical consistency. The tester handles judgment.

## Inputs required

- Draft .feature file(s) from draft-gherkin
- Coverage report from draft-gherkin (regulatory nodes, underspecified flags, gaps)
- PRD (for tracing ambiguities back to their source)
- Solution Detailed Design (for routing design-level ambiguities)

## Process

### Step 1 — Present the coverage report first, not the scenarios

Show the tester the coverage report before showing individual scenarios. This gives the tester a map before they navigate:

```
Regulatory nodes covered: N / M
Underspecified flags: [list with PRD references]
Coverage gaps: [list with KG node IDs]
Incident regression scenarios: [list]
Areas recommended for tester system-knowledge focus: [from draft-gherkin handoff note]
```

Ask: "Before reviewing individual scenarios, do any of the flagged areas immediately signal a problem based on your system knowledge?"

Record any immediate flags — these are high-value because they come before the tester has been anchored by reading the generated scenarios.

### Step 2 — Walk through scenarios by functional area

For each scenario, present it and ask the tester to make exactly one of these calls:

| Decision | Meaning |
|----------|---------|
| ✅ Accept | Scenario is correct and appropriately scoped |
| ✏️ Adjust | Scenario is directionally right but needs modification — tester describes the change |
| ✂️ Split | Scenario is too broad — tester identifies the split boundary |
| 🔀 Merge | Scenario overlaps with another — tester identifies which ones collapse |
| ❌ Remove | Scenario tests something already covered elsewhere or out of scope |
| 🚩 Flag — PRD | Scenario cannot be made concrete because the behavior is underspecified — route back to PRD |
| 🚩 Flag — Solution Design | Scenario cannot be made concrete because the implementation detail is unresolved — route back to Solution Design |
| ➕ Add | System knowledge scenario — tester describes a scenario the LLM could not have known |

Do not ask the tester to write Gherkin. For ✏️ Adjust and ➕ Add, capture the tester's intent in plain language and generate the Gherkin syntax on their behalf for confirmation.

### Step 3 — Capture system knowledge additions

For every ➕ Add decision, ask the tester to describe:
- What condition or interaction their system knowledge reveals
- Which module, component, or flow is involved
- What the expected outcome is

Generate a draft scenario from this description and confirm with the tester before adding it.

Ask: "Is this scenario specific to the current UC, or should it exist as a standing regression scenario that applies more broadly?" If the latter, flag it for KG ingestion as a reusable test pattern.

### Step 4 — Process ambiguity flags

For each 🚩 Flag, determine the routing before moving on:

**Route to PRD when:**
- The behavior itself is not defined (what should happen on this path is not stated)
- The scope is unclear (is this UC supposed to handle this case at all?)
- Success/failure criteria are missing

**Route to Solution Design when:**
- The behavior is defined but the implementation approach is unresolved (which component handles this)
- Shape constraints conflict with the required behavior
- The impact list is incomplete (a component that needs to change is not in scope)

For each flag, produce a routing item:

```
FLAG: <scenario description>
Route to: PRD | Solution Design
Reason: <one sentence — what specifically is missing>
Raised by: Tester system knowledge | LLM underspecification flag
Blocking: Gherkin commit until resolved
```

Collect all routing items. Present them as a batch to send — do not interrupt the review session to resolve them one by one.

### Step 5 — Risk weighting check

After reviewing all scenarios, present the regulatory node coverage map:

> "These regulatory nodes are covered by N scenarios. Based on your knowledge of regulatory risk in this area, are there any nodes where you judge the coverage insufficient?"

The LLM generates proportional coverage based on what it can see. The tester applies risk weighting: high-regulatory-exposure areas may need more scenarios than the behavior specification strictly requires, because the cost of a compliance failure is asymmetric.

If the tester identifies under-covered areas, generate additional scenarios to their description.

### Step 6 — Produce outputs

**Reviewed .feature file(s):** incorporating all accepted, adjusted, split, merged, and added scenarios. Scenarios flagged for removal are excluded. Coverage tags and requirement IDs applied consistently.

**Routing batch:** all 🚩 Flag items formatted for sending to analyst (PRD route) or Developer + Analyst (Solution Design route). These are blockers on scenario commit.

**Commit-ready confirmation:** once all flags are resolved and the tester confirms the scenario set is complete, the scenarios are ready to commit as failing tests. The tester's confirmation is the gate — not the LLM's judgment.

## Hard constraints

- Never commit scenarios while routing flags remain unresolved. The flag batch must be sent and responses received before commit.
- Never write Gherkin in the tester's voice — the tester describes intent, the skill generates syntax, the tester confirms. The tester should never need to type Gherkin manually.
- Route flags correctly. A behavior-undefined flag that gets sent to Solution Design will produce a design change that doesn't fix the problem. Routing matters.
- Do not collapse a tester's ✂️ Split decision by deciding the split is unnecessary. If the tester says split, split.
- System knowledge additions are high-value artefacts. Never skip the question about whether they should be promoted to reusable KG test patterns.
