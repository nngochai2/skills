---
name: check-code-history
description: "Use this skill when a ticket's data and business rules have already been checked (or aren't the question) and the next step is verifying what the code actually does, plus tracing why it was built that way. Triggers: user says 'check the code', 'why was this built this way', or a code comment references an old ticket number (e.g. 'TFS 12345'). Do NOT use this to make code changes — this skill is read-only investigation. Do NOT use this to write to Azure DevOps — ADO access here is read-only by design."
---

> **MCP tools used by this skill:** codegraph (`get_class_overview`, `get_class_dependencies`, `get_transitive_impact`, `find_method_callers`, `get_field_impact` — structural verification), Azure DevOps read-only (`fetch_work_item`, `search_work_items`, `fetch_comments`, `fetch_related_items` — historical context only, never written to)

# Check code + history — verify implementation and trace prior decisions

## Purpose

Confirm what the code actually does (as opposed to what the KG or PL/SQL rules say it should do), and — when a code comment references an old ticket number, or the history behind a piece of logic isn't obvious — search Azure DevOps read-only for the prior work item that explains why it was built that way.

This is the third of three investigative lenses in the ticket-triage workflow (data → business rules → code/history) — see the `ticket-triage` agent for the full sequence.

## Inputs required

- The component/class believed responsible for the behaviour in question
- Any ticket numbers found in code comments (e.g. "TFS 12345", "AB#12345") — these are breadcrumbs, look for them

## Process

### Step 1 — Verify structure with codegraph first

Use the codegraph MCP tools to understand how the component fits together — `get_class_overview` for shape, `get_class_dependencies`/`get_transitive_impact` for blast radius, `find_method_callers`/`get_field_impact` for who reads/writes what. This is fast and reliable for relationship questions.

Codegraph is structural only — it has no method-body data. For "does the code actually do X," read the source directly. Do not stop at codegraph and assume its structural answer covers a behavioral question.

### Step 2 — Look for historical ticket references in the source

While reading source, watch for comments referencing old ticket numbers (legacy "TFS ..." references, or ADO work item IDs like "AB#12345"). These are often the only record of *why* a piece of logic exists.

### Step 3 — Search Azure DevOps (read-only) for context

If a ticket number was found in a comment, fetch it directly with `fetch_work_item`. Pull `fetch_comments` and `fetch_related_items` too — the original requirement is often clarified in the comment thread, not just the description.

If no ticket number was found but the history still isn't obvious, use `search_work_items` with a WIQL `WHERE` clause built from relevant keywords (component name, business term) to look for related past work.

Record:
```
ADO reference found: YES (work item ID) | NO
Original requirement/context: <summary, if found>
Related items: <list, if any>
```

### Step 4 — Reconcile and hand off

State plainly what the code does, and — if found — why it was built that way. This is read-only investigation; it does not conclude bug-or-not on its own. Pass the finding forward to the overall triage verdict.

## Hard constraints

- Never write to Azure DevOps through this skill — no work item creation, comments, or field updates. ADO is a reference source only, per the boundary between it (client-facing) and Jira (internal team tracking).
- Never treat a codegraph structural answer as sufficient evidence for a behavioral claim — read the actual source for that.
- Do not fabricate a ticket number or historical rationale when none is found. State plainly that no historical context exists rather than guessing at intent.
