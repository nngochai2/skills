---
name: check-business-rules
description: "Use this skill when a ticket's data has already been verified (or is not in question) and the next step is checking whether the system's behaviour matches the intended business rule or configuration — as distinct from checking transactional data itself. Triggers: user says 'check the business rules', 'is this expected behaviour', or 'what's the rule here'. Do NOT use this for verifying customer/transactional data values — see check-data for that."
---

> **MCP tools used by this skill:** Document KG (parsed business-rule/config docs — primary source), Oracle (`get_package_source`, `get_view_definition`, `describe_object` — for rules implemented as PL/SQL rather than documented)

# Check business rules — verify expected behaviour against documented and DB-encoded rules

## Purpose

Determine what the system is *supposed* to do in the situation the ticket describes, so that a data or code observation can be judged against the right expectation rather than an assumption. Business rules in this stack live in two places, and both need checking:

1. **Documented rules/config** — parsed into the Document KG.
2. **PL/SQL-encoded rules** — some business logic lives in Oracle stored procedures/packages, not just Java application code or documentation.

This is the second of three investigative lenses in the ticket-triage workflow (data → business rules → code/history) — see the `ticket-triage` agent for the full sequence. Asking a BA verbally is the fallback when neither source resolves the question — that step is human judgment and outside this skill's scope.

## Inputs required

- The specific behaviour/scenario in question (from the ticket, or handed off from `check-data`)
- The component or functional area involved, if known

## Process

### Step 1 — Query the Document KG first

Search the Document KG for business-rule or config documentation relevant to the functional area. This is the primary source — check it before reaching for PL/SQL, since a documented rule is more maintainable evidence than reverse-engineering one from source.

Record what's found:
```
KG rule found: YES | NO
Rule: <what it says, verbatim or closely paraphrased>
Node reference: <KG node ID>
```

### Step 2 — Check for PL/SQL-encoded rules

Whether or not the KG had an answer, check whether the relevant behaviour is also (or instead) implemented in the database:

- Use `describe_object` / `get_view_definition` to see if a view encodes derivation logic relevant to the ticket.
- Use `get_package_source` to pull the actual PL/SQL source of a package/procedure that looks relevant, and read it for the specific condition in question.

Do not assume Java application code is the only place logic can live — if the KG is silent or seems incomplete, check here before concluding the rule doesn't exist.

Record:
```
PL/SQL rule found: YES | NO
Source: <package/procedure name>
Relevant logic: <the specific condition, quoted from source>
```

### Step 3 — Reconcile the two sources

If the KG and the PL/SQL source agree, the rule is well-established — proceed with confidence. If they disagree, or the KG doesn't cover what the PL/SQL source does, flag this explicitly rather than silently picking one:

```
Reconciliation: KG AND DB AGREE | KG AND DB DISAGREE | KG SILENT, DB HAS A RULE | NEITHER HAS AN ANSWER
```

A disagreement between documented and implemented rules is itself a finding worth surfacing to the tech lead or BA, independent of the ticket at hand — it's a KG gap.

### Step 4 — Hand off

If neither the KG nor the PL/SQL source resolves the question, say so plainly and recommend asking a BA verbally — do not guess at intended behaviour from training knowledge or general domain assumptions. Pass the rule (or the gap) forward to the overall triage verdict.

## Hard constraints

- Never assert a business rule from general knowledge or assumption. Only from the KG or from PL/SQL source actually read via the Oracle MCP.
- Never silently prefer one source over the other when they disagree — always report the disagreement.
- This skill only reads and reports; it does not judge whether the discovered rule is itself correct or should change. That's a decision for a human, not this skill.
