# ADLC Output Conventions

Defines where skills write their output files and how they declare those paths. This is the single source of truth for folder structure — individual SKILL.md files reference this document in their Output block.

---

## Folder structure

```
docs/
  <ticket>/
    development/
      decisions-checklist.md     ← grill-me
      prd-draft.md               ← to-prd (local copy before GitLab publish)
      routing-batch.md           ← review-gherkin
    test/
      features/
        <functional-area>.feature  ← draft-gherkin → review-gherkin
        coverage-report.md         ← draft-gherkin
```

Artifacts that do not land in `docs/` (they are published externally):

| Artifact | Destination |
|----------|-------------|
| PRD issue | Jira (published by `to-prd` via MCP, as an Epic) |
| Implementation issues + DAG | Jira (published by `decompose-issues` via MCP, as Stories parented to the PRD Epic) |
| Preflight result | Jira issue description, under a `## Preflight log` heading (written by `preflight-check` via MCP — no comment/note tool exists) |
| KG gap notes, decision notes | Obsidian vault (committed manually after `grill-me` / `annotate-kg`) |

---

## The ticket identifier

`<ticket>` is a short identifier that scopes all artifacts from a single Use Case implementation effort. It is free-form but should be stable across sessions.

Preferred formats (in order):
1. Jira epic key or slug — `PROJ-42` or `epic-42`
2. Use Case code — `UC-014`
3. A short descriptive slug — `einvoicing-vat-recompute`

Skills ask for it with this prompt:

> "What ticket or identifier should I use for this session's output folder? (e.g. `epic-42`, `UC-014`, or a short slug — this becomes `docs/<ticket>/` in the repo)"

---

## Which skills ask for the ticket identifier

Any skill that opens a fresh conversation asks for the ticket at the very start — before any KG queries, before any questions to the user — if it is not already present in the conversation context.

| Skill | Session behaviour | Asks for ticket? |
|-------|-------------------|-----------------|
| `grill-me` | Always starts the chain | Yes — first action |
| `to-prd` | Follows grill-me in same conversation | No — inherits from context |
| `draft-gherkin` | Same or new conversation | Yes if new conversation |
| `review-gherkin` | New conversation recommended | Yes |
| `decompose-issues` | Always fresh conversation | Yes — first action |
| `preflight-check` | Always fresh conversation, per issue | Yes — first action |
| `annotate-kg` | Same or new conversation | Yes if new conversation |

---

## Output block format in SKILL.md files

Each skill that writes local files declares its outputs in a blockquote immediately below the frontmatter, grouped with the MCP tool declaration if one exists. Follow this format exactly:

```
> **Outputs:**
> - `docs/<ticket>/development/decisions-checklist.md`
> - `docs/<ticket>/development/kg-gap-notes.md` *(draft only — committed to Obsidian manually)*
```

For a single output, use the inline form:

```
> **Output:** `docs/<ticket>/development/routing-batch.md`
```

Skills that produce no local files omit the Output block entirely.
