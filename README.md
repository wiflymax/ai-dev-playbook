# AI Development Playbook

A set of documents and disciplines for keeping agent-driven codebases coherent at scale.

This playbook exists because large codebases written primarily by coding agents are at risk of a specific failure mode: **collapse under their own weight.** Past a certain size, the volume of agent-written code outpaces anyone's ability to hold the architecture in their head. Patterns the codebase moved past get re-introduced. Refactors land that rename files instead of moving behavior. Tests keep passing while contracts silently drift. Bug fixes accumulate as duplicate logic rather than consolidating. The codebase technically works until it doesn't, and by then it's hard to recover.

The playbook prevents collapse by giving the codebase a structure that survives agent context limits and session turnover. The mechanisms are:

- **Rule files placed inside each major folder.** Short markdown files (named `AGENTS.md` or `CLAUDE.md` depending on your agent) that list the local rules for that folder: what kinds of code belong there, which patterns are forbidden, which files must not be edited by hand. Agents read these automatically every time they touch the folder, so the rules persist across sessions even when chat history is lost.

- **Plans that define "done" with numbers, not adjectives.** Instead of "consolidate the duplicate logic," a plan says "reduce file X to ≤500 lines and add ≥20 invariant tests." Concrete targets prevent agents from declaring work complete when it merely *looks* complete. A number either gets hit or it doesn't.

- **A required closeout format at the end of every unit of work.** Each unit ends with a short structured report listing what changed, what was verified, what evidence was produced, and a single yes/no answer: "Ready to advance: yes/no." When the answer is no, the agent stops. The format is identical every time, so a different agent picking up later sees exactly where the previous one left off and why.

- **A set of standing project documents for big or risky work.** Parallel rewrites, migrations, and multi-system integrations need more than a single plan — they need a roadmap, a catalog of small commit-sized work units, a contract describing what behavior must be preserved, and a policy describing what can and can't change while the work is in flight. Together these documents resolve every meaningful question in advance, so the agent doing the work has no gaps to guess at. Most hallucination is an agent filling in a gap; this set of documents removes the gaps.

- **A script that grades the project's readiness based on evidence files, not on the agent's word.** For high-stakes work, a small script (run locally or in CI) reads the evidence the project has produced — closeout reports, test outputs, parity comparisons, rehearsal logs — and outputs a structured list of everything still blocking the project from shipping. If anything is missing, faked, or stale, the script refuses to return green. The agent cannot talk past it.

It has been used to maintain a 170k LOC application, feature work and a full language migration of the entire codebase. Both phases — steady-state maintenance and a large parallel rewrite — drove the patterns documented here. The standard tier covers the first; the enforcement-grade tier covers the second.

The playbook is vendor-neutral. It works with Claude Code, Codex, Cursor, Aider, or any custom harness.

## What this is for

This playbook is the right tool when:

- You're running a long-lived agent-driven project (multi-week, multi-session).
- The codebase is large enough that no single session can hold the architecture in context.
- Mistakes have a cost (regressions, customer impact, data loss, hard-to-roll-back changes).
- You want the agent to do the work without the work going sideways.

It is the wrong tool when:

- The project is a small script, a throwaway prototype, or a few hundred lines of new code.
- The work is a single localized fix in a project that doesn't already follow this discipline.
- You're exploring something where the scope is genuinely unknown.

The playbook adds overhead. That overhead pays back at scale. Below the threshold (~10k LOC, beyond one sprint, multiple sessions over time), it costs more than it saves.

## How it prevents collapse

The failure modes that collapse agent-driven codebases are predictable. The playbook addresses each with a specific mechanism:

| Failure mode | Why it happens | What prevents it |
|---|---|---|
| **Bloat by accretion** | No session sees the global picture; each session adds its own helper instead of finding the existing one | Per-directory rule files name the canonical location for shared logic; architecture tests fail builds that introduce duplicates |
| **False completion** | Agents claim "done" based on the work *looking* done; the underlying intent isn't checked | Slice closeouts require evidence files at named paths; the closeout ends with a binding `Ready to advance: yes/no` gate |
| **Silent contract drift** | Tests pass but the contract a consumer depends on has changed | Contract-and-test matrices anchor every surface to a specific test; package-surface snapshot tests fail when private names get exposed or public names disappear |
| **Lost context across sessions** | A new session re-litigates decisions the project has already made | ADRs and the standing project documents persist the decisions; agent rule files force the new session to read them before acting |
| **Hallucination filling planning gaps** | Plans leave questions unanswered; agents fill them silently and sometimes wrongly | Multi-agent plan validation (described below) resolves ambiguity at the planning stage so the executor has nothing to fill in |
| **Drift during long migrations** | The base system keeps evolving; the new system's evidence rots faster than it can be refreshed | Freeze doctrine pauses feature work on migrated surfaces; change-sync policy forces evidence refresh on every allowed change |
| **Self-graded completion at the project level** | The agent declares the whole project ready when it isn't | The readiness script is the authority; it reads evidence files and refuses to return green when anything is missing |

Each mechanism in the table corresponds to a concrete artifact in this repo. The discipline doesn't prevent failure modes from existing — agents will still try to introduce duplication, still try to claim premature completion, still try to fill in gaps. The discipline makes those attempts visible and cheap to reject.

## How to use this

For a **new project**:

1. Read `PRINCIPLES.md` first. Don't skip it. The templates are useless without the principles that motivate them.
2. Install a root `AGENTS.md` (or `CLAUDE.md` if you're on Claude Code) using the template in `AGENT_INSTRUCTIONS_STARTER_PACK.md`.
3. Add per-directory rule files only when a directory has a non-obvious rule agents reliably get wrong. Five to ten files across a large project is the right scale.
4. When you start a non-trivial refactor or feature, use the phase plan template in `PLAN_AND_PROMPT_TEMPLATES.md`. Numerical floors, not prose acceptance.
5. If the project becomes high-stakes (parallel rewrite, schema migration, multi-system integration where rollback is hard), graduate to the enforcement-grade tier: read `HIGH_STAKES_PROJECT_PLAYBOOK.md` and build the readiness script described in `READINESS_TOOL_SPEC.md`.

For an **existing project** that already follows this discipline:

- Read the root agent rule file before any change.
- Follow the slice closeout format.
- Run the readiness script (if one exists) before claiming anything is ready.
- When tests reveal a real production bug, file it and stop — don't fix it in the same session.

## Applicability to smaller applications

The playbook **does not apply to small applications, single-file scripts, or one-shot fixes.**

Specifically:

- **Under ~10k LOC**: don't use this. Write the code, ship it, iterate.
- **A localized fix in a project that doesn't already follow this discipline**: don't introduce the discipline for one fix. Make the fix. If the same area gets fixed three times in three months, *then* consider whether the architecture needs the discipline.
- **A throwaway prototype**: don't use this. The point of a prototype is to learn, not to maintain.

Within a project that *does* follow this discipline, there are also exemptions:

- **Single-file edits, typo fixes, documentation tweaks**: don't write a phase plan. Just fix it.
- **Bug fixes under ~50 LOC**: don't write a phase plan. Just fix it.
- **Following an existing pattern in a domain that already has one**: use the existing pattern, don't re-derive.

The discipline is for the situations where it pays back. Use the lighter pattern (or no pattern) elsewhere. Otherwise the overhead drowns the value, and worse, the discipline gets discredited as bureaucracy.

The root `AGENTS.md` should explicitly list which kinds of changes can skip the heavier flow. Agents reliably honor explicit exemptions; without them, agents apply the full discipline to every typo fix and the project grinds.

## Operating with agents: plan validation until nothing needs to be inferred

The single highest-leverage technique for reducing hallucination is **resolving ambiguity at the plan stage, not at the implementation stage.**

Hallucination is rarely the agent inventing something out of nothing. It's the agent filling in a gap. If the plan leaves a question unanswered, the agent will answer it — silently, confidently, sometimes wrongly. If the plan resolves every question, there's nothing left to fill in.

The pattern that works:

1. **Draft the plan with one agent.** Phase plan, slice catalog, contract matrix — whatever shape fits the work.
2. **Spawn a fresh agent to review it cold.** Not the agent that wrote it. A new session, no context from the drafting conversation. Ask: "What in this plan would force an executor to make a judgment call? What's ambiguous? What's missing?"
3. **Take the review seriously.** Every gap the reviewer names becomes a new gate, a tighter floor, or a clarified requirement in the plan.
4. **Repeat.** Spawn another reviewer. Repeat until the latest reviewer's findings are nitpicks instead of substance.
5. **Then execute.**

The first review will surface the most. By the third, you're polishing. The plan that survives this process leaves the executor with no judgment calls to make — only work to do.

When you skip this step, hallucination shows up as confident wrong choices: an API contract subtly violated, a schema invariant casually broken, a "small simplification" that removes load-bearing behavior. The executor isn't lying; it's filling a gap you left.

A few rules for this workflow:

- **Reviewer agents must be fresh sessions.** Same conversation = same blind spots. Spawn explicitly.
- **Reviewer findings become document changes, not chat commentary.** If the review says "this is ambiguous," the plan has to change. Otherwise the review didn't happen.
- **Stop only when the latest reviewer can't name a substantive gap.** Three rounds is typical; some plans take five.
- **Don't use reviewer agents to grade execution.** That's what the readiness script is for. Reviewer agents validate *plans*; the script validates *evidence*.

This pattern doubles or triples the planning time. It cuts implementation rework by more than that, and — more importantly — it produces outcomes you can trust without re-reading every line of agent-written code.

## The three layers (for high-stakes work)

When the project crosses into high-stakes territory, the playbook requires three layers operating together:

| Layer | Job | Where it lives |
|---|---|---|
| **Agent configuration** | What the agent is allowed to do, what it always reads | Root `AGENTS.md`, settings, hooks |
| **Project documents** | What the work is, what counts as done | Markdown in the repo |
| **Readiness script** | Machine-checks whether claimed progress is real | A script that emits JSON, exits non-zero when blocked |

The agent grades nothing about its own work. The script grades the evidence. The human reads the script's output and decides.

Skipping any one layer reintroduces the failure mode it prevents. See `HIGH_STAKES_PROJECT_PLAYBOOK.md` for the full pattern.

## Files in this repo

| File | Purpose | Tier |
|---|---|---|
| `PRINCIPLES.md` | Load-bearing principles, failure modes, day-zero setup | Standard |
| `PLAN_AND_PROMPT_TEMPLATES.md` | Phase plans, agent briefings, ADRs, high-stakes document set skeletons | Standard + Enforcement |
| `AGENT_INSTRUCTIONS_STARTER_PACK.md` | Templates for `AGENTS.md` / `CLAUDE.md` / per-directory rule files | Standard |
| `HIGH_STAKES_PROJECT_PLAYBOOK.md` | The enforcement-grade pattern for parallel rewrites, migrations, multi-system integrations | Enforcement |
| `READINESS_TOOL_SPEC.md` | Input/output contract and refusal rules for the readiness script | Enforcement |

Reading order: `PRINCIPLES.md` → `AGENT_INSTRUCTIONS_STARTER_PACK.md` → `PLAN_AND_PROMPT_TEMPLATES.md`. If the project is high-stakes, also `HIGH_STAKES_PROJECT_PLAYBOOK.md` → `READINESS_TOOL_SPEC.md` before writing code.

## Closing note

The patterns here aren't theoretical. Each one is the response to a failure that happened — duplication that compounded, refactors that didn't refactor, contracts that drifted, evidence that rotted, agents that closed their own gates. The discipline doesn't prevent failures; it makes them cheap and recoverable.

If you adopt this and find a failure mode the playbook doesn't address, encode the rule that would have prevented it. The playbook should get stricter over time, not looser.
