# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working in this repository.

## What this repository is

A skill (and agent) definition library for an AI-assisted development lifecycle (ADLC). Skills and agents are prompt instruction files — not runnable code. There are no build, test, or lint commands. All work here is editing Markdown files.

The library is organized into four tracks — see `CONTEXT.md` for the canonical definitions:
- **`project`** — the document-driven feature-development pipeline (UC → PRD → design → tests → implementation), run against a consuming codebase
- **`support`** — ad-hoc, instant-use work outside the pipeline (e.g. ticket triage), run against a consuming codebase
- **`common`** — meta-tooling for this library itself (e.g. the discipline for authoring/editing skills here), not run against a consuming codebase
- **`deprecated`** — fully specified but currently unusable due to an external blocker, not a design flaw

`skills/` is split into all four tracks. `agents/` only mirrors `project` and `support` — an agent is a recurring-job orchestrator tied to a consuming codebase's pipeline, and this library has no such recurring job to orchestrate against itself, so there is no `agents/common/`.

## Skill vs agent

A **skill** is a single-purpose instruction module for one step of a workflow — often interactive, requiring human judgment mid-process (e.g. `grill-me`'s interview). Lives at `skills/<track>/<name>/SKILL.md`.

An **agent** is an autonomous, instant-use orchestrator that groups several related skills — and the MCP tool access they need — around one recurring job, so it can be invoked directly instead of running each step by hand. Agents invoke skills; skills do not invoke agents. Lives at `agents/<track>/<name>/AGENT.md`.

When adapting an existing skill, don't reclassify it as an agent (or vice versa) without checking whether its process is genuinely autonomous end-to-end (agent) or requires interactive human judgment mid-process (skill) — see `CONTEXT.md` and the root `README.md`'s provenance table for the reasoning behind each current classification.

## Skill/agent file structure

Each skill lives at `skills/<track>/<name>/SKILL.md` and must have YAML frontmatter at the top:

```markdown
---
name: <skill-slug>
description: "<trigger description — what situation activates this skill, what to NOT use it for>"
---
```

Each agent lives at `agents/<track>/<name>/AGENT.md` with the same frontmatter shape, plus a `tools:` field listing the skills and MCP tool access it needs.

The `description` field is loaded by the Claude Code harness to decide when to invoke a skill or agent. It must be precise about triggers AND non-triggers (the `Do NOT use` clauses).

### Invocation mode

Every `SKILL.md` is either **user-invoked** or **model-invoked** — the axis is who can reach it, following Pocock's v1.1 `.agents/invocation.md` convention:

- **User-invoked** — add `disable-model-invocation: true` to the frontmatter. Reachable only by a human typing the skill's name. Use this for skills that sit at a deliberate pipeline gate and must never fire opportunistically just because a description phrase matched: `grill-me`, `grill-with-docs`, `to-prd`, `decompose-issues`, `annotate-kg`, and all three deprecated skills (`draft-gherkin`, `review-gherkin`, `tdd` — the field also acts as a safety net against accidental auto-invocation of blocked skills). The same field also fits skills a human deliberately reaches for outside any pipeline — `writing-great-skills` (`common` track) should never fire just because a task involves editing a `SKILL.md`.
- **Model-invoked** — omit the field. Reachable by the model autonomously, or by a human typing the name. Use this for skills that hold a reusable investigative or review discipline the agent should reach for the moment a task fits, without waiting to be asked: `code-review`, `check-data`, `check-business-rules`, `check-code-history`, `diagnosing-bugs`.

The test: could the model usefully reach for this on its own, mid-task? If yes, model-invoked. If the skill's value depends on a human deliberately choosing to start it (a tech-lead-gated DAG review, a Developer providing shape constraints, a HITL decision capture), user-invoked.

This field applies to `SKILL.md` only. Agents (`AGENT.md`) stay autonomously reachable by design — an agent's whole purpose is instant, standalone invocation (e.g. `preflight-check` firing as an issue moves to In Progress, `ticket-triage` firing on a pasted ticket), so the user/model split doesn't apply to them the same way.

The body of each SKILL.md/AGENT.md contains:
- **Purpose** — what it produces and why
- **Inputs required** — what must be present before it runs
- **Process** — numbered steps, each with sub-steps
- **Hard constraints** — invariants it must never violate

MCP tools and local outputs are declared in blockquotes at the top of the body (below frontmatter), e.g.:
```
> **Jira MCP tools used by this skill:** `jira_create_issue`, `jira_link_issues`
> **Outputs:**
> - `docs/<ticket>/development/decisions-checklist.md`
```

Skills that produce no local files omit the Output block. See `skills/project/CONVENTIONS.md` for the project track's folder structure, ticket identifier guidance, and output block format rules — the support track has no such convention (see `skills/support/CONVENTIONS.md`); it produces a verdict handed to the human, not a committed file.

`README.md` at the repo root (not `skills/README.md` — there isn't one) holds the full provenance table, workflow maps, and per-skill prerequisites.

## Project track — workflow map

```
Use Case document
      ↓
[ grill-me ]          → decisions checklist + KG gap note drafts
      ↓
[ to-prd ]            → PRD Jira issue, Epic (published via MCP)
      ↓
  Solution Detailed Design (manual, by Dev + Analyst)
      ↓
  (optional, currently deprecated: draft-gherkin → review-gherkin →
   scenarios committed as failing tests, once test team automation is ready)
      ↓
[ decompose-issues ]  → dependency DAG → Jira issues (Story, parented to PRD Epic) with HITL/AFK labels (published via MCP)
      ↓
  Per issue, as it moves to In Progress:
[ preflight-check (agent) ] → routing decision (HITL/AFK) + Copilot context block (AFK only), appended to issue description
      ↓
  HITL branch: Human decision → [ annotate-kg ] → KG gap note committed to Obsidian
  AFK branch:  Copilot + KG context (issue-scoped)
      ↓
  Acceptance criteria verified manually → issue closed
      ↓
[ code-review ] → Standards + Spec findings (parallel sub-agents), reported side by side
      ↓
  Peer review + MR, PRD as anchor → merge
```

`grill-with-docs` (project track) doesn't sit in this fixed sequence — it's invoked whenever a plan needs stress-testing against existing `CONTEXT.md`/ADRs, in a consuming project or on this repo itself.

## Support track — workflow map

```
ServiceDesk ticket (pasted manually — no ServiceDesk MCP)
      ↓
[ ticket-triage (agent) ] → runs check-data, check-business-rules, check-code-history in sequence
      ↓
  Verdict (bug / not-a-bug / data issue / needs-BA-input) + evidence, handed to the human
      ↓
  Verdict = bug: [ diagnosing-bugs ] → root cause + fix + regression test (or documented no-seam finding)
      ↓
  Human decides: reply to customer, log a fix (Jira), or no action
```

Session boundaries and per-skill prerequisites are in the root `README.md`.

## External dependencies skills rely on

Skills reference these external systems via MCP — they are not in this repo:

| Dependency | Purpose | Used by |
|------------|---------|---------|
| Document KG (Neo4j via MCP) | Regulatory nodes, existing scenarios, business-rule docs, incident post-mortems | grill-me, grill-with-docs, preflight-check, annotate-kg, check-business-rules, diagnosing-bugs (regulatory-exposure check before adding prod instrumentation) |
| Code graph (jQAssistant via MCP) | Blast radius, dependency depth, component coupling — structural only, no method bodies | grill-me, grill-with-docs, decompose-issues, preflight-check, check-code-history, diagnosing-bugs (optional) |
| Jira API (via MCP) | Create/update/link/transition issues — no comment/note tool, so traceability is appended to the issue description; `jira_get_issue` for read access | to-prd, decompose-issues, preflight-check, code-review |
| Oracle DB (via MCP) | Read-only data queries (`execute_query`) and PL/SQL source inspection (`get_package_source`, `get_view_definition`) — mutating SQL is rejected by the server | check-data, check-business-rules, diagnosing-bugs (optional, inspection only) |
| Azure DevOps (via MCP, read-only) | Historical ticket context (client-facing tracker) — never written to | check-code-history |
| ServiceDesk | Ticket intake — no MCP; ticket text is pasted into the session manually | ticket-triage |
| Obsidian vault | KG gap note storage | annotate-kg |

## Standards files that skills reference

These files must exist in the project that uses these skills (not in this repo):

| File | Used by |
|------|---------|
| `/docs/standards/gherkin-conventions.md` | draft-gherkin, review-gherkin *(deprecated)* |
| `/docs/standards/prd-template.md` | to-prd, decompose-issues, preflight-check |
| `/docs/standards/issue-template.md` | decompose-issues, preflight-check |
| `/docs/standards/kg-gap-tiers.md` | grill-me, annotate-kg |

## Editing guidance

When updating a skill or agent:
- Keep `Hard constraints` sections as the last section — downstream skills, referenced in the root `README.md`, treat them as invariants
- The `description` frontmatter field drives when the harness invokes the skill/agent; changes here have immediate routing impact
- HITL/AFK routing logic in `decompose-issues` and `preflight-check` must stay consistent with each other — they share the same gateway rules
- The tech stack assumed throughout is Java/Oracle/MuleSoft + Jira (not GitHub/GitLab/TypeScript) — do not introduce GitHub/GitLab-specific tooling or TypeScript patterns
- When adapting a Pocock-derived skill, check the original at `.claude/skills/<name>/SKILL.md` before adding new process formalism to replace something that's been dropped (e.g. a deprecated Gherkin dependency) — default to Pocock's simpler original pattern unless there's a concrete reason to add more structure. **Caveat:** this local snapshot is pinned to Pocock's pre-v1.1 skill set (no `code-review`, `diagnosing-bugs`, `to-spec`/`to-tickets` rename, or primitive decomposition like `grilling`/`domain-modeling`). His repo is now versioned at https://github.com/mattpocock/skills (v1.1.0, July 2026) — check upstream directly for anything not present in `.claude/skills/`, don't assume the local snapshot is current.
- Before recommending a specific MCP tool name, verify it against the actual server implementation in `D:\Cloned Projects\NAA\mcp` rather than assuming parity with a different system's tool set (e.g. Jira's MCP has no comment/note tool, unlike the old GitLab integration)
