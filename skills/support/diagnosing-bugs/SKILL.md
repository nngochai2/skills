---
name: diagnosing-bugs
description: "Use this skill when a bug's root cause needs to be found — after ticket-triage confirms a BUG verdict, or standalone whenever the user says 'diagnose this', 'debug this', or reports something broken/throwing/failing/slow. Do NOT use this to determine whether something is actually a bug in the first place — that's ticket-triage's check-data/check-business-rules/check-code-history skills. This skill assumes the bug is already confirmed and finds why, then fixes it with a regression test."
---

> **MCP tools used by this skill (both optional):** codegraph (`get_class_overview`, `get_class_dependencies`, `find_method_callers` — structural navigation for hypothesis-building), Oracle read-only `execute_query` (inspecting data state — cannot mutate fixtures, `execute_query` is guarded to SELECT-only server-side)

# Diagnosing bugs — root-cause discipline

## Purpose

A discipline for hard bugs: the ones that resist a first glance, the intermittent flake, the regression that crept in between two known-good states. This is the follow-on step after `ticket-triage` returns a `BUG` verdict — that verdict says *something is wrong*; this skill finds *why*, then fixes it with a regression test. Skip phases only when explicitly justified.

When exploring the codebase, read `CONTEXT.md` (if it exists) for a clear mental model of the relevant modules, and check any ADRs in the area being touched.

## Inputs required

- A confirmed bug symptom — from `ticket-triage`'s `BUG` verdict, or reported directly
- Access to the codebase, test runner, and a dev/staging environment where the bug can be exercised

## Process

### Phase 1 — Build a feedback loop

**This is the skill.** Everything else is mechanical. If you have a **tight** pass/fail signal for the bug — one that goes red on *this* bug — you will find the cause; bisection, hypothesis-testing, and instrumentation all just consume it. If you don't have one, no amount of staring at code will save you.

Spend disproportionate effort here. Be aggressive. Be creative. Refuse to give up.

**Ways to construct one — try roughly in this order:**

1. **Failing test** at whatever seam reaches the bug — unit, integration, or a MuleSoft flow test.
2. **Curl / HTTP script** against a running dev server or MuleSoft flow endpoint.
3. **CLI or batch invocation** with a fixture input, diffing output against a known-good snapshot.
4. **Oracle query harness** — run the suspect query or PL/SQL package against a known dataset (via `execute_query`, `get_package_source`) and diff against expected. Since the MCP is read-only, any fixture setup that requires mutating data must be done through normal dev tooling outside the MCP, not this skill.
5. **Replay a captured trace.** Save a real request/payload/event log to disk; replay it through the code path in isolation.
6. **Throwaway harness.** Spin up a minimal subset of the system (one service, mocked deps) that exercises the bug code path with a single call.
7. **Property / fuzz loop.** If the bug is "sometimes wrong output," run many random inputs and look for the failure mode.
8. **Bisection harness.** If the bug appeared between two known states (commit, dataset, config), automate "boot at state X, check, repeat" so it can be bisected mechanically.
9. **Differential loop.** Run the same input through old vs. new code (or two configs) and diff the outputs.
10. **HITL script.** Last resort — if a human must click something, drive them with a scripted checklist so the loop is still structured. Captured output feeds back into diagnosis.

Once you have *a* loop, tighten it: make it faster (skip unrelated init, narrow scope), make the signal sharper (assert on the specific symptom, not "didn't crash"), make it more deterministic (pin time, seed RNG, isolate filesystem/network). A 30-second flaky loop is barely better than no loop; a 2-second deterministic one is a debugging superpower.

**Non-deterministic bugs:** the goal is a higher reproduction rate, not a clean repro. Loop the trigger repeatedly, add stress, narrow timing windows. A 50%-flake bug is debuggable; 1% is not — keep raising the rate until it is.

**When you genuinely cannot build a loop:** stop and say so explicitly. List what was tried. Ask for: (a) access to an environment that reproduces it, (b) a captured artifact (log dump, HAR file, screen recording with timestamps), or (c) permission to add temporary instrumentation. If the code path touches a regulatory node (check the Document KG), temporary production instrumentation needs explicit human sign-off before adding it — do not add it silently. Do not proceed to hypothesise without a loop.

**Completion criterion:** Phase 1 is done when you can name one command — already run at least once, paste the invocation and its output — that is:
- [ ] **Red-capable** — drives the actual bug code path and asserts the exact reported symptom, not "runs without erroring."
- [ ] **Deterministic** — same verdict every run (or a pinned, high reproduction rate for flaky bugs).
- [ ] **Fast** — seconds, not minutes.
- [ ] **Runnable unattended** — a human in the loop only via a scripted HITL checklist.

If you catch yourself reading code to build a theory before this command exists, stop — jumping straight to a hypothesis is the exact failure this skill prevents.

### Phase 2 — Reproduce + minimise

Run the loop. Watch it go red. Confirm the failure mode matches what was reported — not a different failure that happens to be nearby. Wrong bug, wrong fix.

**Minimise:** shrink the repro to the smallest scenario that still goes red. Cut inputs, callers, config, data, and steps one at a time, re-running the loop after each cut — keep only what's load-bearing. Done when every remaining element is load-bearing (removing any one makes the loop go green).

Do not proceed until reproduced *and* minimised.

### Phase 3 — Hypothesise

Generate 3–5 ranked hypotheses before testing any of them. Single-hypothesis generation anchors on the first plausible idea. Each hypothesis must be falsifiable: "If `<X>` is the cause, then `<changing Y>` will make the bug disappear / `<changing Z>` will make it worse." If a prediction can't be stated, the hypothesis is a vibe — discard or sharpen it.

If codegraph MCP is available, use it here to trace caller/dependency chains before manually grepping — it's fast and reliable for narrowing which components are even in play.

Show the ranked list to whoever reported the bug before testing. Domain knowledge often re-ranks it instantly ("we just deployed a change to #3"). Don't block on it if they're unavailable — proceed with the ranking.

### Phase 4 — Instrument

Each probe maps to a specific prediction from Phase 3. Change one variable at a time.

Preference order: (1) debugger/REPL inspection if the environment supports it — one breakpoint beats ten logs; (2) targeted logs at the boundaries that distinguish hypotheses; never "log everything and grep." Tag every debug log with a unique prefix (e.g. `[DEBUG-a4f2]`) so cleanup at the end is one grep.

For Oracle-side instrumentation, the MCP's `execute_query` is read-only — use normal DBA/dev tooling (trace, `DBMS_PROFILER`) outside the MCP if PL/SQL-side instrumentation is needed.

**Perf branch:** for performance regressions, logs usually mislead. Establish a baseline measurement (timing harness, profiler, query plan) first, then bisect. Measure first, fix second.

### Phase 5 — Fix + regression test

Write the regression test **before** the fix — but only if there is a correct seam for it: one where the test exercises the real bug pattern as it occurs at the actual call site, not a shallow seam that gives false confidence.

**If no correct seam exists, that itself is the finding.** Note it and flag it to the team lead — this repo has no dedicated architecture-improvement skill to hand off to yet, so the flag is the deliverable.

If a correct seam exists: turn the minimised repro into a failing test at that seam, watch it fail, apply the fix, watch it pass, then re-run the Phase 1 loop against the original (un-minimised) scenario.

### Phase 6 — Cleanup + post-mortem

Required before declaring done:
- [ ] Original repro no longer reproduces (re-run the Phase 1 loop)
- [ ] Regression test passes (or absence of a correct seam is documented)
- [ ] All `[DEBUG-...]` instrumentation removed (grep the prefix)
- [ ] Throwaway harnesses deleted or clearly marked as debug-only
- [ ] The hypothesis that turned out correct is stated plainly, for the commit message and for whoever logs the follow-up ticket

Hand the outcome back to the human: per the support track's no-automatic-writes convention (`skills/support/CONVENTIONS.md`), logging the fix as a tracked Jira ticket, and any customer-facing follow-up, stays a manual call — this skill does not open or close tickets on its own.

## Hard constraints

- Never skip straight to a hypothesis before Phase 1's red-capable, already-run command exists. This is the discipline the skill exists to enforce.
- This skill writes code — the fix and its regression test. It must never write to Jira, Azure DevOps, or a production system on its own; logging the fix as a tracked ticket and any customer communication stay manual, per the support track's no-automatic-writes convention.
- Never add production instrumentation touching a regulatory-exposed component (check the Document KG) without explicit human sign-off first.
- Do not fabricate a hypothesis's falsifiable prediction just to move forward — discard or sharpen it instead.
- Do not skip Phase 6's checklist. Leftover debug instrumentation and unremoved throwaway harnesses are themselves bugs left behind.
