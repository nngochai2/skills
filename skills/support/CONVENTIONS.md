# Support Track Conventions

Defines how `skills/support/` differs from `skills/project/` in output handling. This is the single source of truth for support-track conventions — individual SKILL.md files reference this document if they need to.

---

## No `docs/<ticket>/` output folder

Unlike the project track (see `skills/project/CONVENTIONS.md`), support skills do not write files to a ticket-scoped output folder. There is no ticket identifier to establish, and no local artefact to commit.

Support skills and agents produce a **verdict handed to the human** — printed in the conversation, not saved to disk. The support engineer decides what to do with it: reply to the customer, log a fix ticket, or take no action. Nothing gets written automatically to any external system either — every MCP tool used across `check-data`, `check-business-rules`, `check-code-history`, and `ticket-triage` is read-only.

## Why no ticket identifier

The project track's `docs/<ticket>/` convention exists because project-stream artefacts (decisions checklists, PRD drafts, feature files) accumulate across a multi-session pipeline and need a stable home to reference back to. Support-track investigations are single-session and self-contained — the ticket text itself (pasted manually, since there is no ServiceDesk MCP) is the only input, and the verdict is the only output. There is nothing that benefits from a stable file path across sessions.

If a future support skill needs to persist something across sessions, revisit this — don't assume the project track's `docs/<ticket>/` pattern applies without checking whether the same justification (multi-session accumulation) actually holds.
