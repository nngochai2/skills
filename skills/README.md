# ADLC Skills

Skills for the AI-assisted development lifecycle (ADLC). Covers the full workflow from Use Case interrogation through KG annotation after implementation.

---

## Provenance

| Skill | Source | Notes |
|-------|--------|-------|
| `tdd` | Matt Pocock — use as-is | Red-green-refactor loop. No adaptation needed. |
| `grill-me` | Adapted from Pocock's `grill-me` + `grill-with-docs` | Rewritten for KG-aware regulatory interrogation and Obsidian/Neo4j pipeline. |
| `to-prd` | Adapted from Pocock's `to-prd` | Adds mandatory shape constraints section, KG node references, GitLab (not GitHub) issue creation. |
| `draft-gherkin` | Net-new | No Pocock equivalent. Compliance-specific: regulatory node coverage, incident-driven scenarios, underspecification flagging. |
| `review-gherkin` | Net-new | No Pocock equivalent. Tester-as-router pattern with ambiguity routing to PRD vs Solution Design. |
| `decompose-issues` | Adapted from Pocock's `to-issues` | Rewritten for coupling-aware dependency DAG, blast radius granularity, GitLab (not GitHub), HITL/AFK labels. |
| `preflight-check` | Net-new | No Pocock equivalent. Code graph pre-flight with gateway rules for regulatory exposure and drift detection. |
| `annotate-kg` | Net-new | No Pocock equivalent. HITL decision capture with tiered KG node classification and wikilink generation. |

Pocock's original skills (`grill-me`, `to-prd`, `to-issues`) are available at https://github.com/mattpocock/skills — MIT licensed. They assume GitHub, TypeScript, and greenfield projects. The adaptations here replace those assumptions with GitLab, Java/Oracle/MuleSoft, and a legacy compliance codebase.

---

## Workflow map

```
Use Case document
      ↓
[ grill-me ]          ← Document KG (MCP), Code graph (MCP, optional)
  Output: decisions checklist + KG gap note drafts
      ↓
[ to-prd ]            ← decisions checklist, Developer shape constraints input
  Output: PRD GitLab issue (behavior spec + scope + shape constraints)
      ↓
  Solution Detailed Design (Dev + Analyst, manual — code graph validates impact)
      ↓
[ draft-gherkin ]     ← PRD, Solution Design, Document KG (MCP)
  Output: draft .feature files + coverage report + underspecification flags
      ↓
[ review-gherkin ]    ← draft .feature files, coverage report, PRD, Solution Design
  Output: reviewed .feature files + routing batch (flags back to PRD / Solution Design)
      ↓
  Scenarios committed as failing tests (manual git commit)
      ↓
[ decompose-issues ]  ← committed .feature files, PRD, Solution Design, Code graph (MCP)
  Output: dependency DAG → GitLab issues with HITL/AFK labels + blocked-by links
      ↓
  GitLab Kanban board (HITL/AFK labels, blocked-by links)
      ↓
  Per issue, as it moves to In Progress:
[ preflight-check ]   ← issue body, Code graph (MCP), Document KG (MCP), Solution Design
  Output: routing decision (HITL/AFK) + Copilot context block (AFK only)
      ↓
  HITL branch: Human decision → [ annotate-kg ] → KG gap note drafted + committed to Obsidian
  AFK branch:  Copilot + KG context (issue-scoped smart zone)
      ↓
  Cucumber scenario passes (done signal per issue)
      ↓
  Code review + MR (peer review, PRD as anchor)
      ↓
  Merge to integration
      ↓
  Jenkins: full Cucumber suite  |  Tester: exploratory testing
```

---

## Prerequisites per skill

### grill-me
- Document KG ingested into Neo4j and accessible via MCP server
- Code graph (optional — enhances blast radius questions; session works without it)
- UC document present in context

### to-prd
- Completed decisions checklist from grill-me in context
- Developer available to provide shape constraints (skill will ask explicitly)

### draft-gherkin
- PRD GitLab issue URL or content in context
- Solution Detailed Design content in context
- Document KG accessible via MCP
- Team Gherkin naming convention document (see `/docs/standards/gherkin-conventions.md`)

### review-gherkin
- draft-gherkin output (.feature files + coverage report) in context
- PRD in context (for ambiguity routing)
- Solution Detailed Design in context (for routing)
- Tester present and engaged — this skill requires live human judgment

### decompose-issues
- Committed .feature files in the repository
- PRD GitLab issue content in context
- Solution Detailed Design in context
- Code graph accessible via MCP (skill degrades gracefully without it — all issues labelled `needs-preflight`)
- GitLab API access configured

### preflight-check
- GitLab issue body in context
- Code graph accessible via MCP and recently rebuilt (jQAssistant run since last merge)
- Document KG accessible via MCP
- Solution Detailed Design in context

### annotate-kg
- Human decision description from the implementer
- Pre-flight output (regulatory nodes in scope) in context
- GitLab issue reference
- Obsidian vault accessible for note commit

### tdd (Pocock — use as-is)
- Failing Cucumber scenario(s) in context or referenced
- Code graph accessible for pattern consistency (optional but recommended)

---

## Standards this skill set depends on

These must exist before the skills produce consistent output. Create them once, reference them in every relevant skill invocation.

| Standard | Location | Used by |
|----------|----------|---------|
| Gherkin naming and tagging convention | `/docs/standards/gherkin-conventions.md` | draft-gherkin, review-gherkin |
| PRD template | `/docs/standards/prd-template.md` | to-prd, decompose-issues, preflight-check |
| Issue body template | `/docs/standards/issue-template.md` | decompose-issues, preflight-check |
| KG gap tier definitions | `/docs/standards/kg-gap-tiers.md` | grill-me, annotate-kg |
| KG node taxonomy (tag controlled vocabulary) | Obsidian Tag hub nodes | draft-gherkin, annotate-kg |

---

## Context window notes

Following Pocock's guidance on session boundaries:

- `grill-me` → `to-prd`: same conversation. The decisions checklist is the handoff.
- `to-prd` → `draft-gherkin`: can be same or new conversation. PRD GitLab issue URL is the portable artefact.
- `draft-gherkin` → `review-gherkin`: new conversation recommended. Feature files and coverage report are the portable artefacts.
- `decompose-issues`: fresh conversation. Inputs are all file/issue references, not prior conversation content.
- `preflight-check`: fresh conversation per issue. Issue body is self-contained.
- `annotate-kg`: can follow preflight-check in the same conversation if context permits.
