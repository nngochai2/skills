# ADLC Skills

Skills (and agents) for the AI-assisted development lifecycle (ADLC). Organized into four tracks — see `CONTEXT.md` at the repo root for the canonical definitions:

- **`project/`** — the document-driven feature-development pipeline, from Use Case interrogation through KG annotation after implementation. For developers on the feature/project stream, against a consuming codebase.
- **`support/`** — ad-hoc, instant-use work that sits outside the project pipeline (e.g. ticket triage). For developers on the support stream, against a consuming codebase.
- **`common/`** — meta-tooling for this library itself (e.g. the discipline for authoring/editing skills here), not run against a consuming codebase.
- **`deprecated/`** — fully specified but currently unusable due to an external blocker, not a design flaw.

`agents/` mirrors only `project/` and `support/` — an agent is a recurring-job orchestrator tied to a consuming codebase's pipeline, and this library has no such recurring job to orchestrate against itself, so there is no `agents/common/`. An **agent** is an autonomous orchestrator that groups several related skills (and the MCP tool access they need) around one recurring job, invoked standalone rather than run step-by-step. A **skill** is a single-purpose step, often interactive.

---

## Provenance

**Invocation** follows Pocock's v1.1 user-invoked/model-invoked split (see `CLAUDE.md`): **User** = reachable only by typing the skill's name (`disable-model-invocation: true`), for skills that sit at a deliberate pipeline gate. **Model** = the agent can also reach for it autonomously, for skills that hold a reusable investigative/review discipline. **Agent** = the invocation-mode field doesn't apply; agents are autonomously reachable by design.

| Skill/Agent | Track | Invocation | Source | Notes |
|-------|-------|-------|--------|-------|
| `grill-me` | project | User | Adapted from Pocock's `grill-me` + `grill-with-docs` | Rewritten for KG-aware regulatory interrogation and Obsidian/Neo4j pipeline. |
| `grill-with-docs` | project | User | Adapted from Pocock's `grill-with-docs` | Generic plan-vs-documentation stress-test, made KG-aware (degrades gracefully without one) and codegraph-aware (structural questions only — source reading still required for behavioral claims). Usable both in consuming project codebases and as a meta-tool for this repo. |
| `to-prd` | project | User | Adapted from Pocock's `to-prd` | Adds mandatory shape constraints section, KG node references, Jira (not GitHub/GitLab) issue creation. |
| `decompose-issues` | project | User | Adapted from Pocock's `to-issues`/`to-tickets` | Rewritten for coupling-aware dependency DAG, blast radius granularity, Jira (not GitHub/GitLab). No longer requires committed Gherkin scenarios — decomposes from PRD + Solution Design directly, using tracer-bullet vertical slices as the unit of work. Reverted to Pocock's plain acceptance-criteria checklist (no Cucumber-based done signal) since that formalism no longer has anything to attach to. Ported the expand→migrate→contract sequencing for wide refactors from Pocock's v1.1 `to-tickets` as an exception to vertical slicing. This is the tracked milestone for measuring AI-assisted delivery in the project stream. |
| `annotate-kg` | project | User | Net-new | No Pocock equivalent. HITL decision capture with tiered KG node classification and wikilink generation. References Jira issue keys, not GitLab. |
| `preflight-check` | **agent**, project | Agent | Net-new | No Pocock equivalent. Code graph pre-flight with gateway rules for regulatory exposure and drift detection. Moved from a skill to an agent — it's rule-based and autonomous until the final routing decision, unlike the interactive skills above. Rewritten for Jira (no comment/note tool — routing changes are appended to the issue description under a `## Preflight log` heading instead). |
| `code-review` | project | Model | Adapted from Pocock's v1.1 `code-review` | Two-axis (Standards + Fowler smell baseline / Spec) parallel-subagent diff review. Spec axis reads a Jira issue via `jira_get_issue` instead of a GitHub issue/PR; shape-constraint violations count as Spec findings since our `to-prd` bakes them into the PRD rather than a separate standards doc. Runs after implementation, before merge — distinct from `preflight-check`'s pre-implementation routing decision. |
| `draft-gherkin` | **deprecated** | User | Net-new | Compliance-specific scenario generation. Blocked: the test team isn't ready to consume committed Gherkin scenarios for automation. Fully specified, not a design flaw. User-invoked doubles as a safety net against accidental auto-invocation while blocked. |
| `review-gherkin` | **deprecated** | User | Net-new | Tester-as-router pattern with ambiguity routing to PRD vs Solution Design. Blocked alongside `draft-gherkin`. |
| `tdd` | **deprecated** | User | Matt Pocock — use as-is | Red-green-refactor loop, TypeScript examples unchanged. Blocked: this project's unit-test tooling is still ad hoc scripts, not a real test runner — no seam to run the loop against yet. Not a design flaw; re-activate (with Java/JUnit examples) once test infrastructure is in place. |
| `check-data` | support | Model | Net-new | Verifies customer/transactional data against a ticket, via the Oracle MCP's read-only `execute_query`. |
| `check-business-rules` | support | Model | Net-new | Verifies expected behaviour against the Document KG and PL/SQL-encoded rules (Oracle `get_package_source`/`get_view_definition`) — business logic in this stack lives in both places. |
| `check-code-history` | support | Model | Net-new | Verifies actual code behaviour via codegraph (structural only) plus read-only Azure DevOps lookups for historical ticket context (e.g. old "TFS ..." references in code comments). |
| `ticket-triage` | **agent**, support | Agent | Net-new | Orchestrates `check-data` → `check-business-rules` → `check-code-history` against a pasted ServiceDesk ticket (no ServiceDesk MCP exists) and produces a bug/not-a-bug verdict. No automatic writes anywhere — closing tickets and any follow-up stays manual. |
| `diagnosing-bugs` | support | Model | Adapted from Pocock's v1.1 `diagnosing-bugs` | Root-cause discipline (build a red-capable feedback loop → reproduce/minimise → hypothesise → instrument → fix → regression test) — the follow-on step once `ticket-triage` returns a `BUG` verdict. Notes the Oracle MCP's read-only constraint on fixture setup and requires human sign-off before adding production instrumentation to a regulatory-exposed component. |
| `writing-great-skills` | **common** | User | Adapted from Pocock's v1.1 `writing-great-skills` (renamed from his pre-v1.1 `write-a-skill`) | Meta-tooling for this repo, not a consuming-codebase skill — the vocabulary and discipline (invocation-mode tradeoffs, information hierarchy, leading words, pruning, failure modes) for authoring or editing any skill/agent here. Kept nearly verbatim; disclosed reference in `GLOSSARY.md`. Consult it, don't reclassify it — it doesn't replace the frontmatter/section-shape rules in `CLAUDE.md`. |

Pocock's original skills are available at https://github.com/mattpocock/skills — MIT licensed (his repo is now versioned; this repo tracks his v1.1.0, July 2026). They assume GitHub, TypeScript, and greenfield projects. The adaptations here replace those assumptions with Jira, Java/Oracle/MuleSoft, and a legacy compliance codebase. Some of his v1.1 skills were deliberately **not** adopted: his tracker-abstraction layer (`setup-matt-pocock-skills`, `docs/agents/issue-tracker.md`) solves multi-tracker portability we don't need (`docs/adr/0001` already commits this repo to Jira as the one real tracker), and his `triage` skill targets externally-reported GitHub issues/PRs with auto-posted comments — a different problem from our `ticket-triage` (ServiceDesk diagnosis, verdict handed to a human, no auto-writes).

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
[ code-review ] ← Jira issue (jira_get_issue), git diff since branch point
  Output: Standards + Spec findings, reported side by side (not merged/reranked)
      ↓
  Peer review + MR, PRD as anchor
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
  Verdict = BUG:
[ diagnosing-bugs ] ← confirmed symptom, codegraph (MCP, optional), Oracle MCP (optional, read-only)
  Output: root cause + fix + regression test (or a documented no-seam finding)
      ↓
  Human decides: reply to customer, log a fix (Jira), or no action — nothing is written automatically
```

Each of the three `check-*` skills is also independently invocable for a support engineer who already suspects a specific cause and only wants one lens checked. `diagnosing-bugs` is likewise standalone-invocable whenever a bug is already confirmed and root-causing is the only question — it doesn't require having gone through `ticket-triage` first.

## Common track

Has no workflow map — it isn't run against a consuming codebase, so it doesn't chain from or into anything in the project/support pipelines. `writing-great-skills` is invoked standalone, whenever this repo's own `SKILL.md`/`AGENT.md` files are being drafted, split, merged, or reviewed for bloat.

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

### code-review
- A fixed point to diff against (branch, commit, tag) — skill will ask if not supplied
- Jira issue key the diff implements (skill will ask; Spec axis is skipped if none exists)
- Git access to the working tree

### check-data / check-business-rules / check-code-history
- Ticket description pasted into the session (no ServiceDesk MCP)
- Oracle MCP, Document KG MCP, codegraph MCP, and/or Azure DevOps MCP (read-only) as relevant to each skill — see each skill's tool declaration

### ticket-triage (agent)
- Same as the three skills above — it orchestrates them and needs the same MCP access as all three combined

### diagnosing-bugs
- A confirmed bug symptom (from `ticket-triage`'s `BUG` verdict, or reported directly)
- Access to the codebase, test runner, and a dev/staging environment where the bug can be exercised
- codegraph MCP and Oracle MCP are both optional — the skill degrades to manual investigation without them

### draft-gherkin / review-gherkin / tdd (deprecated)
- Not currently runnable in this pipeline — see the `Deprecated` note at the top of each SKILL.md

### writing-great-skills (common)
- None — pure reference, invoked by typing its name while editing a skill/agent file in this repo

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

Full details for the project track — including which skills ask for the ticket identifier, which artifacts go to Jira/Obsidian instead, and the Output block format for SKILL.md files — are in [`skills/project/CONVENTIONS.md`](skills/project/CONVENTIONS.md).

Support-track skills do **not** use this convention — they produce a verdict handed to the human, not a committed file. See [`skills/support/CONVENTIONS.md`](skills/support/CONVENTIONS.md).

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
- `code-review`: fresh conversation recommended. Inputs are a git ref and a Jira issue key, not prior conversation content.
- `ticket-triage` (agent) and the three `check-*` skills: standalone per ticket, no session-boundary dependency on anything else in this repo.
- `diagnosing-bugs`: can follow a `ticket-triage` `BUG` verdict in the same conversation if context permits, or start fresh from a confirmed symptom.
- `writing-great-skills`: standalone — invoke whenever authoring or editing a skill/agent in this repo, no session-boundary dependency on anything else.

---

## License

MIT — see [LICENSE](LICENSE). Several skills here (`tdd`, `grill-me`, `grill-with-docs`, `to-prd`, `decompose-issues`, `code-review`, `diagnosing-bugs`, `writing-great-skills`) are adapted from [Matt Pocock's skills](https://github.com/mattpocock/skills), also MIT licensed.
