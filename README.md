# skills

A Claude Code skill (and agent) library for an AI-assisted development lifecycle (ADLC) on a Java/Oracle/MuleSoft compliance codebase, tracked in Jira.

Organized into three tracks:

- **`project`** — the document-driven feature-development pipeline: Use Case interrogation → PRD → Solution Design → issue decomposition → implementation → KG annotation.
- **`support`** — common, ad-hoc, instant-use work outside that pipeline, such as ServiceDesk ticket triage.
- **`deprecated`** — fully specified skills currently unusable due to an external blocker (not a design flaw) — e.g. the Gherkin-generation skills, blocked on the test team's automation readiness.

`agents/` mirrors the same three-way split. A **skill** is a single-purpose, often-interactive step; an **agent** is an autonomous orchestrator that chains several related skills around one recurring job.

See [`skills/README.md`](skills/README.md) for the full provenance table, workflow maps, and per-skill prerequisites. See [`CLAUDE.md`](CLAUDE.md) for the file-structure conventions Claude Code follows when editing this repo. See [`CONTEXT.md`](CONTEXT.md) for the glossary of terms used throughout (project/support/deprecated, skill vs. agent), and [`docs/adr/`](docs/adr) for the reasoning behind hard-to-reverse decisions (e.g. the Jira tracker choice).

Several skills here (`tdd`, `grill-me`, `grill-with-docs`, `to-prd`, `decompose-issues`) are adapted from [Matt Pocock's skills](https://github.com/mattpocock/skills) (MIT licensed), rewritten for a GitLab/GitHub-free, Java/Oracle/MuleSoft, Jira-tracked, KG-aware pipeline. See `skills/README.md`'s provenance table for what changed in each.

## License

MIT — see [LICENSE](LICENSE).
