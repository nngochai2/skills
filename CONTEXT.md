# ADLC Skills Library

This repository defines skills (and agents) used across the AI-assisted development lifecycle. It is organized into three tracks that describe *why a skill exists and who it's for*, not what tool it wraps.

## Language

**Project skill**:
A skill belonging to the document-driven feature-development pipeline — chained from a Use Case document through PRD, design, tests, and implementation. Used by developers on the feature/project stream. Lives under `skills/project/`.
_Avoid_: ADLC skill (too broad — support and deprecated skills are also part of the ADLC)

**Support skill**:
A skill for common, ad-hoc, or instant-use work that sits outside the project pipeline — e.g. troubleshooting — used by developers on the support stream. Lives under `skills/support/`.
_Avoid_: utility skill, common skill

**Deprecated skill**:
A skill that is fully specified but currently unusable because of an external blocker (a team or system it depends on isn't ready), not because the skill itself is flawed. Lives under `skills/deprecated/`.
_Avoid_: retired skill, disabled skill

**Skill**:
A single-purpose instruction module for one step of a workflow. May be interactive, requiring human judgment mid-process (e.g. `grill-me`'s interview, `review-gherkin`'s tester routing), or mechanical, running a repeatable check unattended. Lives under `skills/<track>/`.

**Agent**:
An autonomous, instant-use orchestrator that groups several related skills — and the MCP tool access they need — around one recurring job, so a developer can invoke it directly instead of running each step by hand. Agents invoke skills; skills do not invoke agents. Lives under `agents/<track>/`.
_Avoid_: subagent (an implementation-level term for how Claude Code executes an agent, not a domain concept)
