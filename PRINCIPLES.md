# AI-Directed Development Principles

## The pattern that works

Five steps, in order:

1. **Plan document with measurable acceptance.** Numerical floors, not prose. Per-slice mappings, not "consolidate where appropriate."
2. **Slice into commit-revertible units.** Each slice has its own LOC ceiling, verification command, and acceptance criterion.
3. **Stop on bug discovery.** When a test reveals a real production bug, file it, xfail the test, do not fix in the same context.
4. **Evidence-tied deferral.** When a slice can't hit its floor, the reason cites measurement (clone scanner output, LOC count, named seam), not optimism.
5. **Reporting format with explicit gate.** Each slice closes with a structured report ending in "Ready to advance: yes/no."

When all five hold, refactors land cleanly. When any one fails, the failure compounds across sessions.

## Day-zero setup

Install these before the first feature ships. They cost a day and pay back every refactor.

### 1. Per-directory agent instructions files

Each major directory needs a 5-15 line agent instructions file (`AGENTS.md` is the cross-vendor convention; `CLAUDE.md` for Claude Code) stating the local rules: what belongs here, what doesn't, what patterns are forbidden, what compatibility surfaces survive. Agents read these on every invocation. They survive context window limits in a way conversation does not.

See `AGENT_INSTRUCTIONS_STARTER_PACK.md` for templates and the filename convention per agent runtime.

### 2. CI architecture guardrails

A test file that fails the build when:

- A new top-level module appears in a package marked as closed
- A file exceeds its LOC ceiling
- An import crosses an architectural boundary (e.g., routes importing provider internals directly)
- A new clone group above a threshold appears in the codebase

The clone-detection part is the highest-leverage. An AST-normalized clone scanner committed at day zero with an empty baseline catches copy-paste duplication the day it happens, not the day a future audit finds it.

### 3. Forward-only data migrations from day one

Migrations are immutable once committed. Schema changes are new migrations, not edits to old ones. Data removal uses tombstones, not destructive deletes. This makes rollback tractable and audit trails real.

If you don't need this on day one, you don't need a database. But once you have one, you need this discipline immediately, not after the first production incident.

### 4. Codegen for cross-language contracts

If the system has a typed backend and a typed frontend (e.g., Python + TypeScript), the boundary types are generated, not hand-maintained. Pydantic → TypeScript is one mechanism. OpenAPI → both is another. Pick one before the second route ships.

Hand-maintained cross-language types are the single highest-volume drift bug in AI-written codebases. AI sessions cannot reliably keep two type definitions in sync across separate files.

### 5. ADR directory

`docs/engineering/architecture/adr/` with five initial ADRs:

- The provider/adapter abstraction (or "we have no providers, here's why")
- The data lifecycle model (forward-only, tombstones, revisions)
- The cross-language contract source-of-truth
- The compatibility-shim policy (when shims live, when they die)
- The "no universal base class" principle (or its explicit rejection)

Each ADR is 200-400 words: context, decision, consequences, status. When future AI sessions propose changes, they cite or amend the ADR. When they don't cite it, ask why.

### 6. Audit / scorecard infrastructure

A `docs/engineering/audits/` directory with a `CLEANUP_SCORECARD.md` that tracks measurable progress across phases. Even early in a project this stays mostly empty — but the *location* exists, so future audits land somewhere durable instead of in chat history.

## Architecture decisions that matter most

In rank order by leverage:

### 1. Provider / adapter abstraction (if multi-vendor)

If the system touches more than one external service, the abstraction goes in before the second integration. Putting it in after the second is 3× the cost. Putting it in after the third is 10× the cost.

The abstraction is not necessarily a base class. A `Protocol`-based plug-in with a registry usually works better than `BaseProvider` inheritance. The principle: each integration owns its own lifecycle (plan, apply, diff, registry, artifacts) but the *seam between system and integration* is one named place.

Example from this project: `lib/providers/<name>/` with `plan.py`, `apply.py`, `diff.py`, `registry.py`. Adding a 5th provider is a known cost.

### 2. The publish lifecycle (draft → snapshot → revision → rollback)

If users author state and that state goes to external systems, the authoring layer needs an explicit draft/publish/revision/rollback model. Not as a feature — as data structure.

The discipline:

- Drafts are mutable, snapshots are immutable
- Snapshots are referenced by revision number, not name
- Deletes are tombstones (with a `restore_or_purge` flow), not destructive
- Rollback restores from a snapshot, never reconstructs

Adding this model after authoring is in production requires backfill migrations and is brutal. Adding it at day one is one schema decision.

### 3. Object-first compilation (if the system targets multiple output formats)

If the same logical intent compiles to multiple output formats (e.g., the same ACL targets DNAC + Aruba MC), the compilation is two phases: logical intent → object catalog → format-specific output. Not three. Not one.

The object catalog is the seam. Once you have it, adding a fifth output format is mechanical. Without it, every new output format is a partial rewrite.

### 4. Codegen at every typed boundary

Stated again because it bears repeating. Every place two parts of the system agree on a type, the type is generated from one canonical source. Backend → frontend is the most obvious. Database schema → application models is less obvious but matters as much.

### 5. Per-directory ownership rules in agent instructions files

Agent instructions files (`AGENTS.md` / `CLAUDE.md`) are not documentation. They are agent instruction. Sessions reliably honor them. Document the rule once, every future session honors it without re-deriving.

## The discipline that produces real work

### Plan documents have numerical floors

Every plan slice has at least one number that can be verified:

- "Reduce X to ≤500 LOC" not "consolidate X"
- "Each Central provider module shrinks by ≥100 LOC per seam" not "extract shared helpers"
- "At least 20 invariant tests" not "comprehensive coverage"
- "No function or class definitions remain in `legacy.py`" not "legacy.py is empty-ish"

Prose acceptance ("substantial reduction," "improved coverage," "cleaner separation") consistently produces under-delivery. AI sessions interpret prose generously toward "good enough." Numbers don't get interpreted.

### Slices are commit-revertible

Each plan slice produces one git commit. Multiple slices may share a PR; they never share a commit. The commit boundary is the revert unit. When a future session needs to bisect a regression, the slice-level granularity is what makes the bisect tractable.

Anti-pattern: bundling "Slice 2A + 2B + 2C all done" into one commit. Anti-pattern: branching mid-slice. Anti-pattern: amending commits across slice boundaries.

### Stop on bug discovery

When implementing a test against current behavior reveals that the behavior is wrong:

1. The test is committed as `@pytest.mark.xfail(reason="<one-line>")`
2. The bug is filed (tracker issue or a markdown file in `docs/engineering/audits/OPEN_DESIGN_DECISIONS.md`)
3. The plan continues to the next slice
4. Production code is NOT modified in the current session

This is counterintuitive but load-bearing. Fixing the bug in the same context as discovering it produces commits that mix test additions with production changes — the diff becomes unreviewable, the failure mode unclear, the rollback story compromised. The xfail-and-defer pattern keeps each unit of work scoped to one concern.

When the bug is later fixed, the fix is a separate plan, separate branch, separate PR. The fix's spec is the xfailed test; the fix completes when the xfail flips to passing without modifying the assertion.

This pattern surfaced four real bugs in this project — package surface regression, worker BaseException survival, cancellation/failure terminal-status conflation, revision-conflict response body missing field. Each was fixed in isolation. Each fix was small (≤150 LOC). No fix introduced a regression.

### Evidence-tied deferral

When a slice cannot hit its numerical floor, the deferral reason cites measurement:

- "Phase 3 family seams deferred: remaining clone footprint is below the 1,500 LOC continuation threshold"
- "SNMP shell extraction deferred: named shell candidates are absent"
- "Push routing dedup deferred: documented as output/error-label risk without stronger snapshots"

Not:

- "We'll come back to this"
- "Phase 3 is too complex to finish now"
- "Deferred for future work"

The difference: evidence-tied deferral lets a future session know what would unblock the work. Optimistic deferral becomes "future work" that never lands.

### Reporting format with explicit gate

Every slice ends with a structured report:

```
Slice: [N]
Tests added / LOC changed: [count, vs. ceiling/floor]
Verification output: [paste]
Bugs discovered (if any): [list with xfail commits]
Ready to advance: [yes / no]
```

The "Ready to advance" line is the gate. When the answer is "no," the AI stops and the human reviews. The pattern that fails: AI claims completion in prose and proceeds to the next slice. The pattern that works: explicit yes/no with the human pausing on every "no."

## Failure modes that look like work but aren't

These all appeared in this project. Naming them prevents repetition.

### Rename-only landings

A refactor renames a 3,000-line `core.py` to `coordinator.py`. The audit shows "core.py: 15 LOC" — looks like a 99% reduction. The actual work is zero: the mass moved one name over.

Detection: post-refactor, the audit's measurable target (LOC reduction, function count, clone count) is met technically but the architectural intent is unmet.

Prevention: plan acceptance criteria measure the *destination*, not the *source*. "diff.py ≤800 LOC AND publish.py ≤700 LOC AND rollback.py ≤600 LOC" is harder to game than "core.py is small."

### False splits

A package has `legacy.py` (3,000 LOC, all the behavior) plus `objects.py` (5 LOC, re-exports from legacy). The directory looks domain-split. The architecture is unchanged.

Detection: the alleged split files all have ≤30 LOC and no function definitions.

Prevention: the plan's done definition requires the splitting files to *own behavior*. "objects.py contains real implementations, not 3-line wrappers" with an AST check is the rule that prevents this.

### Catch-all module in a freshly-split package

Phase 4 of the cleanup split three monolith files into per-domain modules. The split produced `building/misc.py` — a catch-all for helpers that didn't fit cleanly. This *was* the accretion vector Phase 4 was supposed to eliminate. The fix re-emerged as the problem.

Detection: any module named `misc.py`, `helpers.py`, `common.py`, `utils.py` at the root of a package claiming to be domain-organized.

Prevention: architecture test that explicitly forbids these filenames in packages where domain-split is the rule. Allowlist for legitimate exceptions with reason comments.

### Wishful invariants

When writing behavioral tests, AI sessions sometimes state invariants that describe what the system *should* do rather than what it does. These tests fail when implemented because they were testing a feature that doesn't exist.

Detection: invariant names like "X enforces Y" where you can't immediately point to the code that enforces Y. Phrases like "the system ensures..." with no traceable enforcement point.

Prevention: invariant inventory before test implementation. The inventory cites file:line for each invariant's enforcement. Invariants without a citation are flagged for human review before tests are written.

### Completeness drift

The new system implements every surface anyone explicitly named in a slice, and silently omits every surface no slice happened to enumerate. The phase reports all close. The contract matrix all passes. The readiness tool says green. Production breaks the first time a user tries a route the old system had and the new system doesn't.

Detection: a diff between the old system's inventory of surfaces (routes, operations, scheduled jobs, message handlers, entities, providers) and the new system's actual exposed surface. Surfaces the diff finds in the old but not the new must either be implemented or explicitly retired with an amendment.

Prevention: every inventory the project captures must be paired with an **equivalence tool** that compares it against the new system's actual surface. The pair runs as a gate the readiness tool consumes. Without the gate, the inventory is decoration — the project has the data to detect completeness drift but no machine check that flags it.

This is the failure mode that catches projects with otherwise excellent discipline. Every gate the project shipped was internally consistent against its own scope; no gate scoped the comparison to "old system's surface vs new system's surface." The slices closed on what they declared; the aggregate gap accumulated silently. The cutover passed. The first production day surfaced 30+ missing routes the UI calls.

For high-stakes work, see `HIGH_STAKES_PROJECT_PLAYBOOK.md` (the "three gate categories" section) and `READINESS_TOOL_SPEC.md` (the `inventory_mismatch_*` refusal codes).

### Package surface regression

A refactor trims a package's `__init__.py` or `legacy.py` barrel. A private name that other code imported from the package surface gets dropped. Tests fail; production breaks.

Detection: import error from a package after a refactor, citing a name that previously worked.

Prevention: package surface snapshot tests. Enumerate every name imported from the package by other code; allowlist file enforces those names remain exposed. This caught `_ACL_ACTIONS` and similar regressions during cleanup.

### Overclaiming completion based on prose

A phase report says "Phase 3 complete." The scorecard says "Phase 3 complete." The plan has 8 seams; the implementation added strategy contract types (one seam) and didn't extract the executors (the other seven).

Detection: completion claim does not cite measurable evidence. Phrases like "materially improved," "addresses the core issue," "covers the high-priority cases."

Prevention: scorecard rows distinguish "Complete" from "Partial." Each phase has a list of slices. Completion claims cite which slices shipped. "Phase 3 partial: 1 of 8 seams shipped, 7 deferred with documented reasons" is the correct framing if only one seam shipped.

### Compact barrel breaking surface

Trimming a `legacy.py` barrel from 50 lines of explicit `from .x import name1, name2` to 10 lines of `from .x import *` drops every name not in `__all__`. If the target modules don't define `__all__`, this silently breaks anything depending on the now-hidden private names.

Detection: tests fail after a barrel-trim refactor with `ImportError` on a name that previously worked.

Prevention: the rule that explicit re-export ≤30 LOC is acceptable. The rule that `import *` is acceptable only when target modules define `__all__`. State both in the cleanup plan.

## Cross-AI verification

When one AI's audit underperforms, run a second one. This conversation produced two duplication audits with different results: a Claude audit found ~400 reducible LOC; a Codex audit found ~9,500-14,000. The Codex audit was correct.

The reason: prose-based pattern-matching AI agents systematically under-count exact clones in code that looks slightly different at the variable-name level. AST-normalized clone scanners (which Codex used) catch what prose-matching agents miss.

The pattern:

1. First AI does a prose-style audit
2. Second AI runs an AST-normalized clone scanner before any prose analysis
3. Compare results
4. If they disagree by an order of magnitude, the AST-based result is the working baseline
5. The prose audit becomes context, not source of truth

When to do this: any cleanup-class refactor where duplication is the target. Once per refactor cycle, not every session.

## How to know when you're done

AI-written codebases at this scale carry approximately 5-10% reducible duplication. Not 30-50%. The popular narrative about "AI verbosity tax" exceeds the measurement.

Specifically in this project: an 87,500 LOC Python codebase had ~9,500-14,000 LOC of safely-removable duplication (10-16%) plus ~8,000 LOC of structural relocation (legacy.py → named modules). After cleanup, the codebase is ~82,000 LOC. The remaining mass is real feature surface, not waste.

Signs that you're done cleaning up:

- The clone scanner reports remaining clones below a threshold that consolidation would push under
- The largest files are large because they do real work, not because they're catch-alls
- The architectural seams are visible — refactoring one seam doesn't cascade
- A second AI auditor finds no major new clusters

Signs that you're not done:

- Files named `misc.py`, `helpers.py`, `common.py` in packages claiming to be domain-organized
- Top-level barrels with broad imports of "everything from below"
- Cross-cutting concerns scattered across 10+ files instead of one named location
- Tests pass but the docs say the architecture is in transition

The cleanup discipline ends when measurement says it ends, not when it feels done.

## The human role

The discipline depends on the human being the load-bearing reviewer at specific gates. The gates are:

1. **Plan approval before execution.** The plan exists in a markdown file; the human reads it cold and approves the numerical floors before any code is touched.

2. **Bug-discovery pause.** When a slice reveals a bug, the AI stops and the human decides: fix-now-in-new-branch, defer-with-xfail, or design-decision-needed.

3. **Phase closeout review.** The structured report at the end of each phase is read against the plan's done definition. Mismatch produces a "reopen" decision.

4. **Cross-AI audit when stakes are high.** Before committing to a multi-week cleanup, a second AI auditor verifies the scope.

5. **Compatibility/policy decisions.** Whether to retain a shim, whether to add a feature flag, whether a behavior change is acceptable.

The human is not writing code. The human is making the decisions that the AI cannot make from inside its context window. Specifically: cross-session memory, cross-tool verification, scope-vs-effort tradeoffs, and design-decision adjudication when invariants conflict.

## Scaling to high-stakes work

The principles above are sufficient for most agent-driven refactors and feature work. They are not sufficient when the work is *both* large in scope and irreversible in impact — parallel rewrites, schema migrations, multi-system integrations where a bad outcome cannot be cleanly undone.

For that tier, an enforcement-grade pattern adds three layers on top of these principles:

1. **A multi-document project skeleton** (roadmap, slice catalog, contract/test matrix, change-sync policy, per-phase plans) that makes the work executable by agents across sessions without re-deriving scope.
2. **A machine-checkable readiness tool** that emits structured JSON and refuses to declare the project ready when evidence is missing, faked, or stale.
3. **An irreversible-action rule** that prevents the agent from ever closing gates that require production writes, schema changes, or external-system mutation on its own confidence.

The pattern is documented in `HIGH_STAKES_PROJECT_PLAYBOOK.md` with the tool contract in `READINESS_TOOL_SPEC.md`. The principles in this document are the foundation; the playbook is the next tier up. Use the playbook when the cost of a wrong outcome exceeds the overhead of building the document skeleton and the readiness tool — typically multi-week migrations, multi-vendor integrations, or any work where rollback requires more than reverting commits.

## When this discipline doesn't apply

- Projects below ~10k LOC. The overhead exceeds the value. Write the code, ship it, iterate.
- One-shot scripts. No future sessions; no cross-session discipline needed.
- Throwaway prototypes. The point is to learn, not to maintain.
- Solo human authors at small scale. The discipline addresses cross-session AI coherence; a single human can hold the picture in their head.

The discipline scales upward. As projects grow, the rules become more important, not less.

## Summary

Plan with numbers. Slice with commit boundaries. Stop on discovery. Defer with evidence. Report with gates. Trust the measurement over the prose. Run a second auditor when stakes are high. Hold the line on compatibility surfaces. The patterns are not opinions; they are the responses to specific failure modes that this project has hit and resolved.
