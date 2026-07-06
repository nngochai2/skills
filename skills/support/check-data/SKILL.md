---
name: check-data
description: "Use this skill when investigating a ServiceDesk ticket and the first thing to establish is whether the customer's underlying data is correct. Triggers: user says 'check the data', 'is this a data issue', or pastes a ticket description and wants to start the triage from data verification. Do NOT use this to modify data — this skill is read-only by design. Do NOT use it to check business-rule/config tables that govern behaviour (see check-business-rules) — this is for transactional/customer data specifically."
---

> **MCP tools used by this skill:** Oracle (`execute_query` — read-only, mutating SQL is rejected by the server itself; `describe_object`, `search_objects` to locate the right tables/views when the ticket doesn't name them explicitly)

# Check data — verify transactional/customer data against a ticket

## Purpose

Given a ServiceDesk ticket describing something the customer thinks is wrong, determine whether their underlying data actually supports that claim, before assuming it's a code bug or a business-rule question. Most "the system is broken" reports resolve here: the data itself is missing, malformed, or not what the customer believes it is.

This is the first of three investigative lenses in the ticket-triage workflow (data → business rules → code/history) — see the `ticket-triage` agent for the full sequence.

## Inputs required

- The ticket description (pasted manually — there is no ServiceDesk MCP)
- Enough identifying information to locate the relevant row(s): customer ID, invoice number, date range, or similar

## Process

### Step 1 — Identify what data the ticket depends on

Read the ticket and name the specific record(s) it's about. If the ticket is vague ("my invoice is wrong"), ask the user for the identifying value (invoice number, customer ID) rather than guessing at a table.

### Step 2 — Locate the relevant tables/views

If the table or view name isn't already known, use `search_objects` and `describe_object` to find it. Do not guess a table name and query it blind — confirm the schema shape first, especially column names and types, since a `LONG`/precision mismatch or a wrong join can silently produce a plausible-looking wrong answer.

### Step 3 — Query and compare against the ticket's claim

Use `execute_query` (SELECT only — the server rejects anything else) to pull the actual row(s). Compare field-by-field against what the customer describes or expects.

Record the result plainly:
```
Data check: MATCHES CUSTOMER'S EXPECTATION | CONTRADICTS TICKET | DATA MISSING | DATA MALFORMED
Evidence: <the specific field(s) and value(s) that support this conclusion>
```

### Step 4 — Hand off

This skill's job ends at a data verdict — it does not decide whether a data discrepancy is a bug, a business-rule outcome, or expected behaviour. That's what `check-business-rules` and `check-code-history` are for. Pass the verdict and evidence forward.

## Hard constraints

- Never write, update, or delete data. This skill is investigation-only — the Oracle MCP enforces this at the server level (mutating SQL raises an error), but do not attempt to route around it (e.g. via a stored procedure call) even if asked.
- Never conclude "this is a bug" from a data check alone. A data-level contradiction could still be correct behaviour given a business rule you haven't checked yet.
- Cap query results to what's needed to verify the ticket — don't dump entire tables when a targeted `WHERE` clause answers the question.
