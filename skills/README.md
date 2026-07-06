# ADLC Skills

Skills (and agents) for the AI-assisted development lifecycle (ADLC). Organized into three tracks — see `CONTEXT.md` at the repo root for the canonical definitions:

- **`project/`** — the document-driven feature-development pipeline, from Use Case interrogation through KG annotation after implementation. For developers on the feature/project stream.
- **`support/`** — common, ad-hoc, instant-use work that sits outside the project pipeline (e.g. ticket triage). For developers on the support stream.
- **`deprecated/`** — fully specified but currently unusable due to an external blocker, not a design flaw.

`agents/` mirrors the same three-way split. An **agent** is an autonomous orchestrator that groups several related skills (and the MCP tool access they need) around one recurring job, invoked standalone rather than run step-by-step. A **skill** is a single-purpose step, often interactive.

---

## Provenance

| Skill/Agent | Track | Source | Notes |
|-------|-------|--------|-------|
| `tdd` | project | Matt Pocock — use as-is | Red-green-refactor loop. No adaptation needed. |
| `grill-me` | project | Adapted from Pocock's `grill-me` + `grill-with-docs` | Rewritten for KG-aware regulatory interrogation and Obsidian/Neo4j pipeline. |
| `grill-with-docs` | project | Adapted from Pocock's `grill-with-docs` | Generic plan-vs-documentation stress-test, made KG-aware (degrades gracefully without one) and codegraph-aware (structural questions only — source reading still required for behavioral claims). Usable both in consuming project codebases and as a meta-tool for this repo. |
| `to-prd` | project | Adapted from Pocock's `to-prd` | Adds mandatory shape constraints section, KG node references, Jira (not GitHub/GitLab) issue creation. |
| `decompose-issues` | project | Adapted from Pocock's `to-issues` | Rewritten for coupling-aware dependency DAG, blast radius granularity, Jira (not GitHub/GitLab). No longer requires committed Gherkin scenarios — decomposes from PRD + Solution Design directly, using tracer-bullet vertical slices as the unit of work. Reverted to Pocock's plain acceptance-criteria checklist (no Cucumber-based done signal) since that formalism no longer has anything to attach to. This is the tracked milestone for measuring AI-assisted delivery in the project stream. |
| `annotate-kg` | project | Net-new | No Pocock equivalent. HITL decision capture with tiered KG node classification and wikilink generation. References Jira issue keys, not GitLab. |
| `preflight-check` | **agent**, project | Net-new | No Pocock equivalent. Code graph pre-flight with gateway rules for regulatory exposure and drift detection. Moved from a skill to an agent — it's rule-based and autonomous until the final routing decision, unlike the interactive skills above. Rewritten for Jira (no comment/note tool — routing changes are appended to the issue description under a `## Preflight log` heading instead). |
| `draft-gherkin` | **deprecated** | Net-new | Compliance-specific scenario generation. Blocked: the test team isn't ready to consume committed Gherkin scenarios for automation. Fully specified, not a design flaw. |
| `review-gherkin` | **deprecated** | Net-new | Tester-as-router pattern with ambiguity routing to PRD vs Solution Design. Blocked alongside `draft-gherkin`. |
| `check-data` | support | Net-new | Verifies customer/transactional data against a ticket, via the Oracle MCP's read-only `execute_query`. |
| `check-business-rules` | support | Net-new | Verifies expected behaviour against the Document KG and PL/SQL-encoded rules (Oracle `get_package_source`/`get_view_definition`) — business logic in this stack lives in both places. |
| `check-code-history` | support | Net-new | Verifies actual code behaviour via codegraph (structural only) plus read-only Azure DevOps lookups for historical ticket context (e.g. old "TFS ..." references in code comments). |
| `ticket-triage` | **agent**, support | Net-new | Orchestrates `check-data` → `check-business-rules` → `check-code-history` against a pasted ServiceDesk ticket (no ServiceDesk MCP exists) and produces a bug/not-a-bug verdict. No automatic writes anywhere — closing tickets and any follow-up stays manual. |

Pocock's original skills (`grill-me`, `to-prd`, `to-issues`, `grill-with-docs`) are available at https://github.com/mattpocock/skills — MIT licensed. They assume GitHub, TypeScript, and greenfield projects. The adaptations here replace those assumptions with Jira, Java/Oracle/MuleSoft, and a legacy compliance codebase.

---

## Project track — workflow map

```
Use Case document
      ↓
[ grill-me ]          ← Document KG (MCP), Code graph (MCP, optional)
  Output: decisions checklist + KG gap note drafts
      ↓
[ to-prd ]            ← decisions checklist, Developer shape constraints input
  Output: PRD Jira issue (Epic) — behavior spec + scope + shape constraints
      ↓
  Solution Detailed Design (Dev + Analyst, manual — code graph validates impact)
      ↓
  (optional, currently deprecated: draft-gherkin → review-gherkin → scenarios
   committed as failing tests, once the test team is ready for automation)
      ↓
[ decompose-issues ]  ← PRD, Solution Design, Code graph (MCP)
  Output: dependency DAG → Jira issues (Story, parented to the PRD Epic) with HITL/AFK labels + blocked-by links
      ↓
  Jira board (HITL/AFK labels, blocked-by links)
      ↓
  Per issue, as it moves to In Progress:
[ preflight-check agent ] ← issue body, Code graph (MCP), Document KG (MCP), Solution Design
  Output: routing decision (HITL/AFK) + Copilot context block (AFK only), appended to the issue description
      ↓
  HITL branch: Human decision → [ annotate-kg ] → KG gap note drafted + committed to Obsidian
  AFK branch:  Copilot + KG context (issue-scoped)
      ↓
  Acceptance criteria verified manually (no automated done-signal) → issue closed
      ↓
  Code review + MR (peer review, PRD as anchor)
      ↓
  Merge to integration
```

`grill-with-docs` doesn't sit in this fixed sequence — invoke it whenever a plan (at any stage) needs stress-testing against existing `CONTEXT.md`/ADRs, in a consuming project or on this repo itself.

## Support track — workflow map

```
ServiceDesk ticket (pasted manually — no ServiceDesk MCP)
      ↓
[ ticket-triage agent ] ← Oracle MCP, Document KG (MCP), codegraph (MCP), Azure DevOps (MCP, read-only)
      ├─ [ check-data ]           — is the customer's data correct?
      ├─ [ check-business-rules ] — what should happen, per the KG and/or PL/SQL?
      └─ [ check-code-history ]   — what does the code do, and why was it built that way?
  Output: verdict (bug / not-a-bug / data issue / needs-BA-input) + evidence
      ↓
  Human decides: reply to customer, log a fix (Jira), or no action — nothing is written automatically
```

Each of the three skills above is also independently invocable for a support engineer who already suspects a specific cause and only wants one lens checked.

---

## Prerequisites per skill

### grill-me
- Document KG ingested into Neo4j and accessible via MCP server
- Code graph (optional — enhances blast radius questions; session works without it)
- UC document present in context

### grill-with-docs
- Works against any repo with (or without) a `CONTEXT.md`/`docs/adr/` — creates them lazily on first resolved term/ADR
- Document KG and codegraph MCP are both optional — degrades gracefully to local-file-only / source-reading-only if unreachable

### to-prd
- Completed decisions checklist from grill-me in context
- Developer available to provide shape constraints (skill will ask explicitly)
- Jira project key for the PRD Epic

### decompose-issues
- PRD (Jira issue key or content) in context
- Solution Detailed Design in context
- Code graph accessible via MCP (skill degrades gracefully without it — all issues labelled `needs-preflight`)
- Jira MCP access configured, plus the Jira project key and PRD epic key

### preflight-check (agent)
- Jira issue body in context
- Code graph accessible via MCP and recently rebuilt (jQAssistant run since last merge)
- Document KG accessible via MCP
- Solution Detailed Design in context

### annotate-kg
- Human decision description from the implementer
- Pre-flight output (regulatory nodes in scope) in context
- Jira issue reference
- Obsidian vault accessible for note commit

### tdd (Pocock — use as-is)
- Failing test(s) in context or referenced
- Code graph accessible for pattern consistency (optional but recommended)

### check-data / check-business-rules / check-code-history
- Ticket description pasted into the session (no ServiceDesk MCP)
- Oracle MCP, Document KG MCP, codegraph MCP, and/or Azure DevOps MCP (read-only) as relevant to each skill — see each skill's tool declaration

### ticket-triage (agent)
- Same as the three skills above — it orchestrates them and needs the same MCP access as all three combined

### draft-gherkin / review-gherkin (deprecated)
- Not currently runnable in this pipeline — see the `Deprecated` note at the top of each SKILL.md

---

## Output folder structure

Project-track skills write their local output files under `docs/<ticket>/` in the consuming project. The ticket identifier is a short, stable slug chosen at the start of each session (e.g. `epic-42`, `UC-014`).

```
docs/
  <ticket>/
    development/        ← decisions-checklist.md, prd-draft.md
    test/
      features/         ← .feature files, coverage-report.md (deprecated pipeline only)
```

Full details for the project track — including which skills ask for the ticket identifier, which artifacts go to Jira/Obsidian instead, and the Output block format for SKILL.md files — are in [`project/CONVENTIONS.md`](project/CONVENTIONS.md).

Support-track skills do **not** use this convention — they produce a verdict handed to the human, not a committed file. See [`support/CONVENTIONS.md`](support/CONVENTIONS.md).

---

## Standards this skill set depends on

These must exist before the skills produce consistent output. Create them once, reference them in every relevant skill invocation.

| Standard | Location | Used by |
|----------|----------|---------|
| Gherkin naming and tagging convention | `/docs/standards/gherkin-conventions.md` | draft-gherkin, review-gherkin *(deprecated)* |
| PRD template | `/docs/standards/prd-template.md` | to-prd, decompose-issues, preflight-check |
| Issue body template | `/docs/standards/issue-template.md` | decompose-issues, preflight-check |
| KG gap tier definitions | `/docs/standards/kg-gap-tiers.md` | grill-me, annotate-kg |
| KG node taxonomy (tag controlled vocabulary) | Obsidian Tag hub nodes | draft-gherkin *(deprecated)*, annotate-kg |

---

## Context window notes

Following Pocock's guidance on session boundaries:

- `grill-me` → `to-prd`: same conversation. The decisions checklist is the handoff.
- `to-prd` → `decompose-issues`: can be same or new conversation. PRD Jira issue key is the portable artefact.
- `decompose-issues`: fresh conversation recommended. Inputs are all file/issue references, not prior conversation content.
- `preflight-check` (agent): fresh conversation per issue. Issue body is self-contained.
- `annotate-kg`: can follow a HITL preflight result in the same conversation if context permits.
- `grill-with-docs`: standalone — no fixed predecessor/successor, invoke whenever a plan needs stress-testing.
- `ticket-triage` (agent) and the three `check-*` skills: standalone per ticket, no session-boundary dependency on anything else in this repo.
