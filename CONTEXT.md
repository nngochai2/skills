# ADLC Skills Library

This repository defines skills (and agents) used across the AI-assisted development lifecycle. It is organized into four tracks that describe *why a skill exists and who it's for*, not what tool it wraps.

## Language

**Project skill**:
A skill belonging to the document-driven feature-development pipeline — chained from a Use Case document through PRD, design, tests, and implementation. Used by developers on the feature/project stream, against a consuming codebase. Lives under `skills/project/`.
_Avoid_: ADLC skill (too broad — support and deprecated skills are also part of the ADLC)

**Support skill**:
A skill for ad-hoc, instant-use work that sits outside the project pipeline — e.g. troubleshooting — used by developers on the support stream, against a consuming codebase. Lives under `skills/support/`.
_Avoid_: utility skill, common skill (that term now names a distinct track — see below)

**Common skill**:
A skill that is meta-tooling for this library itself, rather than something run against a consuming codebase's project or support stream — e.g. the discipline for authoring or editing skills in this repository. Lives under `skills/common/`. Has no `agents/common/` counterpart: agents are recurring-job orchestrators tied to a consuming codebase's pipeline, and this library has no such recurring job to orchestrate against itself.
_Avoid_: meta skill (accurate but reads as jargon), utility skill (collides with the deliberately avoided term under support skill)

**Deprecated skill**:
A skill that is fully specified but currently unusable because of an external blocker (a team or system it depends on isn't ready), not because the skill itself is flawed. Lives under `skills/deprecated/`.
_Avoid_: retired skill, disabled skill

**Skill**:
A single-purpose instruction module for one step of a workflow. May be interactive, requiring human judgment mid-process (e.g. `grill-me`'s interview, `review-gherkin`'s tester routing), or mechanical, running a repeatable check unattended. Lives under `skills/<track>/`.

**Agent**:
An autonomous, instant-use orchestrator that groups several related skills — and the MCP tool access they need — around one recurring job, so a developer can invoke it directly instead of running each step by hand. Agents invoke skills; skills do not invoke agents. Lives under `agents/<track>/`.
_Avoid_: subagent (an implementation-level term for how Claude Code executes an agent, not a domain concept)
