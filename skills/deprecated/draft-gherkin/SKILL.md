---
name: draft-gherkin
description: "DEPRECATED — do not invoke. Use this skill when a PRD and Solution Detailed Design both exist and Gherkin scenarios need to be generated. Triggers: user says 'draft gherkin', 'generate scenarios', 'write the feature files', or 'draft the tests'. Both PRD and Solution Detailed Design must be present. Do NOT generate scenarios from PRD alone — the Solution Detailed Design's impact list is required for accurate coverage."
disable-model-invocation: true
---

> **Deprecated:** the test team is not yet ready to consume committed Gherkin scenarios for automation. This skill is fully specified and works — it's blocked by an external dependency, not a design flaw. `decompose-issues` no longer requires its output; re-activate this skill (move it back to `skills/project/` and drop the `DEPRECATED` marker below) once the test team's automation pipeline is ready.
>
> **Outputs:**
> - `docs/<ticket>/test/features/<functional-area>.feature`
> - `docs/<ticket>/test/features/coverage-report.md`

# Draft Gherkin — LLM-generated scenario drafts from UC + design + KG

## Purpose

Generate Gherkin scenario drafts that a tester can review, adjust, and route — not write from scratch. The LLM handles mechanical consistency (naming conventions, tagging, requirement linking). The tester handles judgment (system knowledge, granularity, risk weighting).

A vague scenario is a failure. If a requirement cannot be made concrete and unambiguous, flag it as underspecified rather than produce a placeholder.

## Inputs required

- PRD (GitLab issue or pasted content) — provides behavior specification and KG node references
- Solution Detailed Design — provides impact list (components in scope) and shape constraints
- Document KG accessible via MCP

## Process

### Step 0 — Establish the ticket identifier

If a ticket identifier is not already established in the conversation context, ask the user before doing anything else:

> "What ticket or identifier should I use for this session's output folder? (e.g. `epic-42`, `UC-014`, or a short slug — this becomes `docs/<ticket>/` in the repo)"

Record the response as `<ticket>` and use it consistently in all output file paths for this session.

### Step 1 — KG query (do before writing a single scenario)

Query the Document KG via MCP for three things:

**1. Regulatory nodes that must have coverage**
Find all KG regulatory nodes referenced in the PRD. For each: this node must have at least one scenario that exercises the rule it governs. These are mandatory coverage items — gaps here are not acceptable.

**2. Existing scenarios from related areas**
Find Gherkin scenarios in the KG from functionally adjacent areas. Use these for:
- Naming consistency (follow the same Given/When/Then vocabulary where possible)
- Gap detection (if a related area has a scenario for an edge case, check whether this UC needs an equivalent)
- Anti-duplication (do not generate a scenario that already exists and applies unchanged)

**3. Incident post-mortems in blast radius**
Find any incident post-mortems linked to components in the Solution Detailed Design impact list. Each post-mortem represents a known failure mode — generate a scenario that would have caught it, named with the incident reference.

### Step 2 — Map requirements to scenarios

For each item in the PRD Behavior Specification:

a. **Identify the concrete pre-condition, trigger, and outcome.** If any of these three cannot be stated specifically — if the PRD says "handle errors appropriately" without defining what appropriately means — this requirement is underspecified. Do not write a scenario. Go to Step 2c.

b. **Write the scenario.** Follow this format exactly:

```gherkin
@<regulatory-area-tag> @<requirement-id> @<risk-tier>
Scenario: <imperative description of what is being verified>
  Given <specific system state, not vague setup>
  When <specific action with specific inputs>
  Then <specific, verifiable outcome — not "it succeeds">
```

Naming rules:
- Scenario name: imperative verb + object + condition. "Invoice with missing tax ID is rejected at submission" not "Test invalid invoice"
- Given: specific data state. "a draft invoice with tax ID absent" not "an invoice exists"
- When: specific action with specific input. "the invoice is submitted via the eInvoicing API with payload X" not "it is processed"
- Then: specific, verifiable assertion. "the API returns HTTP 422 with error code MISSING_TAX_ID" not "an error is returned"

c. **Flag underspecified requirements.** Write a flag entry, not a scenario:

```
UNDERSPECIFIED: <PRD section reference>
Reason: <what is missing — pre-condition not defined / outcome not specified / error path ambiguous>
Route to: PRD (behavior not defined) | Solution Design (implementation detail not resolved)
```

### Step 3 — Check mandatory regulatory coverage

After drafting all scenarios: cross-reference against the mandatory regulatory node list from Step 1.

For each regulatory node without scenario coverage: write a coverage gap entry:
```
COVERAGE GAP: <KG node ID> — <rule description>
No scenario exercises this rule. Either: (a) the PRD scope intentionally excludes this rule — confirm with tech lead, or (b) a scenario is missing — add one.
```

Do not silently leave a regulatory node uncovered.

### Step 4 — Add incident-driven scenarios

For each incident post-mortem found in Step 1: generate a regression scenario if one does not already exist. Tag it with the incident reference:

```gherkin
@regression @incident-<ID> @<regulatory-area-tag>
Scenario: <what the incident revealed must not recur>
```

### Step 5 — Produce output

**Draft .feature file(s):** organised by functional area, following team naming convention `<uc-id>_<functional-area>.feature`.

**Coverage report:**
```
Regulatory nodes covered: N / M
Underspecified requirements: [list]
Coverage gaps: [list]
Incident regression scenarios added: [list]
```

**Tester handoff note:**
List the areas where system knowledge is most likely needed — components with complex interactions, flows with known history, areas where the KG has sparse coverage. This is what the tester should focus their review on, not the entire scenario set.

**Saving outputs:** save `.feature` files to `docs/<ticket>/test/features/<functional-area>.feature` and the coverage report to `docs/<ticket>/test/features/coverage-report.md`.

## Hard constraints

- Never write a vague scenario. "Given an invoice exists, When it is processed, Then it succeeds" is a failure — treat it as an underspecification flag, not a scenario.
- Every scenario must carry a `@<requirement-id>` tag linking it to a PRD section.
- Every scenario must carry a `@<regulatory-area-tag>` if it exercises a KG regulatory node.
- Regulatory node coverage gaps must be surfaced explicitly — never silently accepted.
- Do not invent regulatory rules not present in the KG. If coverage seems to require a rule you cannot find in the KG, flag it as a KG gap.
- Do not copy-paste scenarios from related areas without verifying they apply to this UC's specific data context and component scope.
