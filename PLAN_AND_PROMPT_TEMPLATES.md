# Plan And Prompt Templates

Templates for the documents the discipline depends on. Each template is opinionated — the sections exist because removing them caused failures in real refactors.

## Phase Plan Template

For any non-trivial refactor or coverage-expansion phase. The plan exists before any code is written.

```markdown
# [Phase Name] Plan

Status: planned | active | complete | blocked | archived

Date: YYYY-MM-DD

Parent: [link to roadmap or audit that motivated this phase]

## Purpose

One paragraph stating what this phase delivers in measurable terms.
"Reduce X to Y" or "Add Z tests covering W." Not "improve Q."

## Scope

In scope:
- [specific files/modules/concerns]

Out of scope:
- [explicit exclusions; this section prevents scope creep]

## Non-Goals (when ambiguity is likely)

For phases where the boundary is contested:
- "Do not introduce a universal base class"
- "Do not modify production code in this phase; only tests"
- "Do not delete compatibility shims; preserve until [later phase]"

## Baseline

Current measurements that the phase will change:

| Metric | Baseline value |
|---|---:|
| [file] LOC | X |
| [test] count | Y |
| [clone scanner] clone groups | Z |

The numbers in this table are the floor against which slices are measured.

## Execution Discipline

- Each slice lands as a separate commit.
- Each slice's verification commands must pass before commit.
- A slice may close at >=90% of its floor only when a documented Stop Condition blocks the remaining gap.
- [Phase-specific rules go here — e.g., "do not modify production code," "compatibility shim X must survive this phase"]

## Slice [Phase]-1: [Name]

Tasks:

- [Specific, file:line-citable task]
- [Specific task]

Numerical floor:

| Metric | Minimum |
|---|---:|
| [thing measured] | [value] |
| [thing measured] | [value] |

Implementation notes:

- [Anything non-obvious about how the slice should land]
- [Anti-patterns specific to this slice]

Verification:

```bash
[commands to run before commit]
```

Acceptance:

- [Bulleted statement, each measurable]
- [Bulleted statement, each measurable]

## Slice [Phase]-2: [Name]

[Same structure]

## Whole-Phase Verification

[Commands that must pass before the phase is marked complete]

## Stop Conditions

Stop and reassess if:

- [Specific condition that indicates the plan's assumptions are wrong]
- [Specific condition that indicates scope creep]
- [Specific condition that indicates a real bug was discovered]

## Done Definition

This phase is done only when:

- [Measurable statement]
- [Measurable statement]
- [Measurable statement]

Each item is verifiable by a command, not by inspection.

## Out Of Scope (Reminder)

Reiterate the most important out-of-scope items. AI sessions executing the plan
often expand scope unless reminded.
```

### Notes on the template

- **The Baseline section is load-bearing.** Without it, the slices have no anchor for their numerical floors. Take the time to measure baseline before writing the slices.

- **Numerical floors must be per-slice, not just per-phase.** A "Phase total ≥2,500 LOC reduction" is gameable across slices. A per-slice floor with an aggregate floor on top is harder to game.

- **Implementation notes per slice** are where you record the non-obvious bits an executor would otherwise have to ask about. The more specific, the less ambiguity.

- **Stop Conditions are not catch-all.** Each one names a specific signal. "If anything goes wrong" is not a stop condition. "If a helper body differs semantically between copies, stop and reclassify the clone family" is a stop condition.

- **Done Definition entries each map to a command.** If you can't write a verification command for a Done Definition item, the item is not measurable.

## Bug-Fix Plan Template

For production fixes triggered by bug discovery. Shorter than a phase plan because the spec is the test that revealed the bug.

```markdown
# [Bug Name] Fix Plan

Status: planned | active | complete

Date: YYYY-MM-DD

Parent context: triggered by [test file or audit reference] which produced
[X] xfail tests / [Y] decision gap proving [the bug].

## The xfail tests are the spec

The [N] tests in [file] describe required behavior:

- test_name_1
- test_name_2

When this plan is complete, these tests flip from xfail to passing without
modifying the assertions.

## Scope

In scope:
- [files modified]

Out of scope:
- [explicit exclusions]

## Non-Goals

- [Things this fix should not touch — usually the long list]

## Per-[Concern] Decision Table

The load-bearing artifact. For each variant of the failing case, state the
exact intended behavior.

| Variant | Behavior | Reason |
|---|---|---|
| [case] | [what happens] | [why] |
| [case] | [what happens] | [why] |

Implementation must match this table exactly.

## Execution Discipline

- Total production-code change <= [N] LOC. If diff approaches this, stop.
- Each slice lands as a separate commit.
- [Production-code-specific rules]

## Slice [W]-1: Archaeology

Tasks:
- Read the relevant code and document existing patterns.
- Produce an alignment doc at [path].
- No code changes in this slice.

## Slice [W]-2 through [W]-N: Implementation

[Per-slice structure as in phase plan]

## Slice [W]-Final: Flip xfail markers

Tasks:
- Remove @pytest.mark.xfail decorators from the spec tests.
- Run; tests now pass.
- Update the parent audit/plan status.

## Manual Verification (if relevant)

Some bugs cannot be verified by pytest alone (signals, processes, UI):

- [Specific manual step]
- [Specific manual step]

## PR Description Template

The PR for this fix must include:
- The decision table from above
- The spec citation (the xfail tests)
- Manual verification results if applicable
- Test plan output

## Done Definition

- All xfail tests pass without xfail markers
- Production LOC change <= [N]
- Manual verifications passed
- Parent audit/plan status updated
```

## AI Executor Briefing Template

The prompt you paste to Codex or Claude to execute a plan. Self-contained — the executor will not have access to this conversation.

```
[Mission statement: one sentence stating what this prompt does.]

SOURCE OF TRUTH (read in order):
1. [path to plan document]
2. [path to audit if applicable]
3. [path to source code under modification]
4. [path to tests that constitute the spec]
5. [any other required reading]

EXECUTION ORDER (strict — do not reorder, do not skip):

Slice [N] → Slice [N+1] → ... → [stop point or completion]

[If multi-phase: name the stop points and what must be true at each gate.]

PER-SLICE RULES:

1. Each slice lands as its own git commit. The commit boundary is the revert unit.
2. Before claiming a slice complete, run the verification commands listed in the plan. Paste the actual output.
3. A slice may close at >=90% of its numerical floor only when a documented Stop Condition blocks the remaining gap. Below 90%, the slice fails and reopens.
4. [Phase-specific rules — e.g., "If a test fails because a monkeypatch path changed, fix the export, not the test."]
5. [Anti-pattern reminder if the phase has a specific failure mode in its history.]

ANTI-PATTERNS — DO NOT DO THESE:

- [Specific named anti-pattern from the plan]
- [Anti-pattern that prior sessions hit and resolved]
- Do not refactor outside the scope of the active slice. If you see other duplication or smells while working, record them in [tracker file] as a new task — do not fix them inline.
- Do not modify production code if this is a test-coverage phase.
- Do not "fix" tests that reveal real bugs. xfail them and continue.

SPECIFIC GUIDANCE FOR [SLICE N]:

[Detailed bullets for the slice's specific approach. Cite file:line where applicable.]

CRITICAL CORRECTNESS DETAIL (if any):

[If there's a subtle ordering, idempotency, or transaction issue that the
executor must get right, spell it out with code-shape examples.]

BRANCH SETUP:

  git switch main && git pull --ff-only  # or appropriate base
  git switch -c [branch-name]

Confirm before starting which base is correct:
  git log main..[parent-branch] --oneline | head -20

REPORTING FORMAT (per slice):

  Slice: [N]
  [Per-slice numerical fields specific to the phase, e.g., LOC change, tests added, etc.]
  Verification output: [paste]
  Bugs discovered (if any): [list]
  Ready to advance: [yes / no]

CLOSEOUT REPORT (after final slice):

  Phase: [Name]
  Slices landed: [SHA list]
  Slices deferred: [list with reasons]
  Acceptance criteria met: [yes / partial / no, with evidence]
  Verification output: [paste full suite]
  Scorecard updated: [yes / no]
  Ready to advance: [yes / no]

If "Ready to advance" is "no" at any point, stop and report.

START WITH [SLICE N]. The first action is [specific concrete first step].
```

### Notes on the briefing template

- **The "Source of Truth" list is ordered.** The executor reads in that order. Put the plan first, the spec second, the code third.

- **"Anti-Patterns — DO NOT DO THESE"** is the most important section. AI sessions reliably honor explicit prohibitions. The patterns to list are the ones that broke before in this codebase.

- **"Critical Correctness Detail"** is for the one subtle thing the executor will get wrong without explicit guidance. Idempotency ordering, transaction boundaries, record-before-reraise patterns, exception inheritance hierarchies. Spell it out with code-shape examples.

- **"Reporting Format"** with explicit `Ready to advance: yes/no` is the gate. Without this, the executor will claim completion in prose and proceed.

- **The "START WITH" instruction** anchors the executor. Without it, the executor sometimes batches or reorders slices.

## ADR Template

```markdown
# ADR-NNNN: [Title]

Status: proposed | accepted | superseded by ADR-MMMM

Date: YYYY-MM-DD

## Context

[2-4 paragraphs. What problem motivated this decision. What constraints
exist. What was considered.]

## Decision

[1-2 paragraphs. The decision itself, stated as a present-tense statement
of fact. "We use codegen for the backend → frontend type boundary."]

## Consequences

Positive:
- [What this decision makes easier]

Negative:
- [What this decision makes harder]

Neutral but worth knowing:
- [What this decision implies that's non-obvious]

## Status

[Current state. If superseded, link to the superseding ADR.]
```

### Notes on ADRs

- Five ADRs at project start is enough. Add more only when a decision keeps being relitigated.
- ADRs are short. 200-400 words is right. If yours is longer, it's probably documentation of an implementation, not a decision.
- Negative consequences are not optional. Every decision has them. Stating them honestly prevents the "why did we decide that?" conversation 6 months later.

## Cleanup Audit Template

For phases where the scope is "find what needs cleaning up" before "clean it up."

```markdown
# [System] Cleanup Audit

Date: YYYY-MM-DD

Scope: [explicit file/directory list]

## Executive Verdict

| Category | Defensible estimate | Confidence | Meaning |
|---|---:|---|---|
| Direct removal | X-Y LOC | High/Med/Low | [definition] |
| Structural relocation | X-Y LOC | High/Med/Low | [definition] |
| [other categories] | X-Y LOC | ... | ... |

[1-paragraph summary stating the highest-leverage finding.]

## Method

[Brief description of what was scanned and how. Cite tools used.]

## Finding 1: [Name]

Severity: Critical | High | Medium | Low

Estimated [duplicate footprint / structural cleanup]: X LOC

Files:
- [file]
- [file]

### Evidence

[Concrete, citation-heavy. File:line references. Numerical measurements.]

### Why this happened

[Diagnosis. Useful for prevention.]

### Safe Fix

[The recommended approach. Steps, ordering, what to preserve.]

### Tests

[Commands to run to verify the fix.]

### Risk

[Real assessment. "Medium-high. The behavior is provider-critical, and
tests monkeypatch historical import paths." Not "Low" by default.]

## Finding 2-N: [...]

## Not Real Duplication

[Patterns that appeared in the scan but shouldn't be treated as bloat.
Document why each is acceptable.]

## Phased Implementation Roadmap

| Phase | Effort | Why |
|---|---|---|
| Phase 1 | X days | [reason for ordering] |
| Phase 2 | Y days | ... |

[For each phase: a slice-level breakdown with floors.]

## Bottom Line

[2-3 paragraphs. The honest summary. What the responsible numbers are.
What the highest-leverage actions are. Where this audit's estimates are
strong vs. weak.]
```

## High-stakes project document set

For the enforcement-grade pattern in `HIGH_STAKES_PROJECT_PLAYBOOK.md`, you need eight documents. Skeletons follow. Each skeleton is intentionally minimal — fill with project-specific content.

### `00_FEASIBILITY_STATEMENT.md`

```markdown
# Feasibility Statement

## Decision

[One paragraph: proceed / proceed with conditions / do not proceed, and why.]

## What this protects

- [Product behaviors that must not break]
- [Data integrity invariants]
- [Customer-facing contracts]

## Risk surface

- [Highest-risk subsystems, ranked]
- [Why each is risky]

## Non-negotiable constraints

- [Constraint that cannot be relaxed without restarting feasibility review]
- [Constraint]

## Recommended posture

- [Parallel vs. in-place, single cutover vs. gradual, etc.]
- [Justification in terms of risk surface above]

## Complexity assessment

| Area | Risk | Notes |
|---|---|---|
| [area] | low/med/high/very high | [why] |
```

### `01_ROADMAP.md`

```markdown
# Roadmap

## Strategy

[One paragraph: how the work proceeds at a high level.]

## Roadmap traceability index

| Phase | Phase plan | Slice IDs | Primary evidence gate |
|---|---|---|---|
| Phase 0: [name] | phases/PHASE_0_*.md | P0-S1 through P0-SN | [what proves this phase done] |
| Phase 1: [name] | phases/PHASE_1_*.md | P1-S1 through P1-SN | [evidence gate] |
| [...] | | | |

## Program acceptance gates

[The list of conditions that must all be true before the project can ship.]

## Phase N: [Name]

Goal: [one sentence]

Deliverables:
- [item]

Exit gate:
- [stated in evidence terms — what file or report proves this phase done]
```

### `02_SLICE_CATALOG.md`

```markdown
# Slice Catalog

## Global slice rules

- [Rules that apply to every slice]

## Slice closeout format (binding)

```
Slice: PN-SM
Roadmap phase:
Phase plan:
Files changed:
Tests added/updated:
Verification output:
Evidence files:
Bugs discovered:
Ready to advance: yes/no
```

## Phase N: [Name]

| Slice | Name | Deliverable | Primary verification |
|---|---|---|---|
| `PN-S1` | [name] | [what evidence file lands] | [command] |
| `PN-S2` | [name] | [deliverable] | [verification] |

Stop conditions:
- [condition]

Required evidence:
- [evidence file path]
```

### `03_METHOD_OF_PROCEDURE.md`

```markdown
# Method of Procedure

## Preconditions

- [What must be true before any slice can start]

## Roles

- Implementer: [responsibilities]
- Parity reviewer: [responsibilities]
- Safety reviewer: [responsibilities]
- Operator: [responsibilities]

## Standard slice procedure

### 1. Define scope
### 2. Capture baseline
### 3. Implement candidate
### 4. Run local checks
### 5. Run baseline tests
### 6. Run parity harness
### 7. Evidence mode
### 8. Record readiness

## Provider/external-system addendum (if applicable)

## Rollback procedure

## Slice report template

[The binding closeout format from the slice catalog]
```

### `04_CONTRACT_AND_TEST_MATRIX.md`

```markdown
# Contract and Test Matrix

## Required commands

```bash
[command 1]
[command 2]
```

## [Surface name, e.g. API and type contract]

| Surface | Required parity | Existing anchors |
|---|---|---|
| [route inventory] | [what must match] | [test file paths] |

## [Surface name, e.g. Auth, session, headers]

| Surface | Required parity | Existing anchors |
|---|---|---|

[... one section per major contract surface ...]

## Migration discipline

| Gate | Required evidence | Anchor |
|---|---|---|
| Typestate/invariant model | [what proves it exists and is followed] | [file] |
| Test portability | [what proves it] | [file] |
| Feature freeze | [what proves it active] | [file] |

## Cutover acceptance

[Final checklist before the project can ship.]
```

### `05_CHANGE_SYNC_AND_POLICY.md`

```markdown
# Change Sync, Schema, and Cutover Policy

## Change synchronization

[Base system can continue to change, but evidence must stay current. List
which behaviors trigger evidence refresh requirements.]

## Schema policy (if applicable)

[Who owns the schema during the migration; when ownership can transfer.]

## Phase N+ feature freeze

Starting at Phase N, [surfaces] are feature-frozen.

Allowed:
- Security fixes
- Production bug fixes
- Fixture-only changes

Not allowed:
- New write semantics
- [other category]

Not frozen: [list surfaces explicitly excluded from freeze]

## Production canary / irreversible-action semantics

[The exact authorization markers required.]

## Cutover rehearsal

[Minimum rehearsal contract.]

## AI-agent guardrails

AI conversion agents must never:
- [explicit prohibition]
- [explicit prohibition]

AI conversion agents must:
- [explicit requirement]
```

### `phases/PHASE_N_*.md`

```markdown
# Phase N: [Name]

Status: planned | active | complete | blocked

Date: YYYY-MM-DD

Roadmap anchor: [link]

Slice catalog: [link]

Contract/test matrix rows:
- [row]
- [row]

## Purpose

[One paragraph]

## Scope

In scope:
-

Out of scope:
-

## Baseline

| Metric | Baseline value |
|---|---:|

## Execution discipline

- [phase-specific rules]

## Slices

| Slice | Name | Required output |
|---|---|---|
| `PN-S1` | [name] | [output] |

## Numerical floors

| Slice | Minimum floor |
|---|---:|
| `PN-S1` | [floor] |

## Whole-phase verification

```bash
[commands]
```

## Stop conditions

- [condition]

## Done definition

This phase is done only when:
- [item]

## AI executor brief

SOURCE OF TRUTH (read in order):
1. [link]
2. [link]

ANTI-PATTERNS:
- [explicit prohibition]

REPORTING FORMAT:

```
Slice: PN-Sx
[per-slice fields]
Ready to advance: yes/no
```
```

### `development-guide/PRINCIPLES.md` (project-specific overlay)

```markdown
# [Project] Development Principles

## The pattern that works (for this project)

[Project-specific instantiation of the general principles in
docs/engineering/ai-development-guide/PRINCIPLES.md]

## Behavior is the contract

[What this project must preserve.]

## Crate / module boundaries

| Module | Owns | Must not own |
|---|---|---|

## [Other project-specific disciplines]

## Completion standard

A phase is complete only when:
- Every slice has a closeout report
- All matrix gates pass or have approved waivers
- Drift is classified
- Readiness tool exits zero for this phase
- Phase plan says Ready to advance: yes
```

## How to use these templates

Order of operations for a new phase:

1. Write the plan document (Phase Plan or Bug-Fix Plan).
2. Read it cold the next day. Revise.
3. Write the AI executor briefing prompt that references the plan.
4. Paste the prompt to the AI executor.
5. Review the per-slice reports as they come in. Pause on any "Ready to advance: no."
6. At phase close, verify the Done Definition is met.

When not to use these templates:

- Single-file refactors. The overhead exceeds the value.
- Bug fixes that take <50 LOC. Just fix it.
- Exploratory work where the scope is genuinely unknown.
- One-shot scripts.

When to amend these templates:

- After a phase reveals a new failure mode worth documenting.
- When a new tool (clone scanner, AST checker, etc.) becomes available.
- When you discover that a template section is consistently ignored — either remove it or make it load-bearing.

The templates are not law. They are the responses to specific failure modes. New failure modes deserve new sections.
