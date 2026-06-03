# Agent Instructions Starter Pack

Agent instructions files are markdown files at known paths that coding agents read on every invocation. They survive context window limits in a way conversation does not.

This pack covers the general pattern and provides templates. The pattern is vendor-neutral; only the *filename* differs across agent runtimes.

## Filename convention by agent runtime

| Runtime | Root file | Per-directory file |
|---|---|---|
| Cross-vendor convention (preferred for new projects) | `AGENTS.md` | `AGENTS.md` |
| Claude Code | `CLAUDE.md` | `CLAUDE.md` |
| Cursor | `.cursorrules` | per-directory not standard |
| Aider | `.aider.conf.yml` references docs | reference docs anywhere |
| Codex CLI / Codex agents | `AGENTS.md` | `AGENTS.md` |
| Custom harnesses | per-harness; usually configurable | per-harness |

**Recommendation for new projects: use `AGENTS.md`.** It is the emerging cross-vendor convention and is read by the most agent runtimes. If you also use Claude Code, point `CLAUDE.md` at `AGENTS.md` with a one-line stub:

```markdown
# Project guidance for Claude Code

This project uses AGENTS.md as the canonical agent instructions file.
Read AGENTS.md before any change.
```

For brevity, the templates below say `AGENTS.md`. Substitute your runtime's filename if different.

## When to use directory-level instructions files

Put an instructions file in a directory when at least one of these is true:

- The directory has a non-obvious rule that agents reliably get wrong without guidance.
- The directory has a public/private contract surface that must not silently drift.
- The directory has a specific naming convention or file-shape constraint.
- The directory has compatibility shims that must survive certain refactors.

Do not put an instructions file in every directory. Five to ten files across a large project is the right scale. Each one must earn its place by encoding a real rule.

## What makes an instructions file effective

- Short. 5–15 bullets total. Files over 30 lines lose signal.
- Imperative. "Do not edit X" not "X should generally not be edited."
- Cites the failure mode when non-obvious. "Compatibility barrels must not accumulate behavior" is good; adding "(this is how god modules return after a refactor)" is better.
- Lists what belongs and what doesn't. Both are useful.
- Names specific files when the rule is file-specific.

## Templates

Each template below is a starting point. Adapt to your project's specific conventions. Remove rules that don't apply; add rules that do.

### Root `AGENTS.md` (project root)

The root file is read most often. Put the highest-leverage cross-cutting rules here. For high-stakes projects, this is also where you point at the readiness tool and the project document set.

```markdown
# Project Guardrails

## Required reading before non-trivial work
1. docs/<roadmap-or-overview>.md
2. docs/<slice-catalog-or-active-plan>.md
3. The active phase plan under docs/.../phases/
4. docs/engineering/ai-development-guide/PRINCIPLES.md

## Hard rules
- The readiness tool (tools/check_readiness.py) is authoritative. Never claim a
  slice or phase is ready while it reports blockers.
- Slice closeouts use the exact template in the slice catalog. The closeout
  ends with "Ready to advance: yes/no". When "no", stop.
- Numerical floors over prose acceptance. "Reduce X to <=Y" not "consolidate X."
- Each slice lands as a separate git commit. Do not batch slices into one commit.
- When tests reveal a real production bug, file it (xfail with reason) and continue.
  Do not fix in the same session as the test discovery.
- Never edit generated files (e.g., generated API types) by hand. Regenerate
  via the named generator script.
- Never take an irreversible action (production write, schema change, force push,
  external message, lab apply against real systems) without explicit human
  authorization recorded in evidence.
- Architecture tests are product guardrails, not optional lint. Update them only
  with an explicit ownership reason.

## When in doubt
Ask. Don't infer.
```

### `lib/AGENTS.md` (library root, if applicable)

For projects where the bulk of code lives in a `lib/` directory. The rules prevent accretion at the package root.

```markdown
# lib/ Guardrails

- Do not add new top-level lib/*.py modules unless the architecture-boundary
  test is updated with an explicit ownership reason.
- Domain code belongs in subpackages (e.g., lib/acl/, lib/catalog/, lib/db/).
- Compatibility barrels may re-export names but must not accumulate behavior.
  A barrel with a function or class definition is a barrel that has lost its purpose.
- Prefer small domain helpers over universal base classes. The universal base
  class is almost always wrong.
- Do not create misc.py, helpers.py, common.py, or utils.py files at the root
  of a domain package. If a function does not fit cleanly into a named domain
  module, the domain split needs revision.
```

### `lib/db/AGENTS.md` (database/data layer)

For projects with a relational database and a domain data layer.

```markdown
# lib/db/ Guardrails

- Database intent code is forward-only. Use tombstones/archive rows for removals
  rather than destructive deletes unless a purge path explicitly owns it.
- Cross-domain workspace helpers belong in a designated cross-domain file,
  not inside a domain package.
- Existing migrations are immutable once committed. Add a new migration for
  schema changes.
- Domain services should not import API routes, external adapters, or worker
  orchestration.
```

### `lib/db/publish/AGENTS.md` (lifecycle layer, if applicable)

For projects with a draft/publish/revision/rollback lifecycle.

```markdown
# lib/db/publish/ Guardrails

- Lifecycle ownership is explicit:
  - coordinator.py is compatibility/orchestration only. No behavior.
  - publish.py owns snapshot/publish behavior.
  - rollback.py owns rollback restore behavior.
  - diff.py owns draft diff behavior.
  - yaml_import.py (or equivalent) owns import sessions.
  - domains/*.py own per-domain helpers — real behavior, not 3-line stubs.
- Do not import publish lifecycle internals from outside lib/db/publish/.
  Use the package's public surface.
- Snapshot serialization must be deterministic. No timestamps, UUIDs, or
  ordering-dependent values in serialized output without explicit sanitization.
```

### `lib/providers/AGENTS.md` (external-adapter layer)

For projects with multiple external integrations (vendors, services, APIs).

```markdown
# lib/providers/ Guardrails

- No universal BaseProvider class. Shared code must be family-scoped or
  generic utility code, not a cross-provider inheritance tree.
- Each provider package owns its lifecycle: plan.py, apply.py, diff.py,
  registry.py, artifacts.py. The mass belongs in these files; do not copy
  lifecycle helpers across them.
- Provider-family packages are for genuinely shared behavior across related
  providers. Do not call something "consolidation" unless both providers'
  modules visibly shrink (>=100 LOC each per seam).
- Provider models.py files own DTOs, constants, and typed exceptions only.
  They must not drag in lifecycle imports (argparse, asyncio, API clients).
- Provider lifecycle modules should not import route, service, UI, or worker code.
```

### `tests/AGENTS.md`

```markdown
# tests/ Guardrails

- Architecture tests are product guardrails, not optional lint.
- Keep fake-client provider tests deterministic. Live external calls are
  not allowed in CI.
- When refactoring internals that tests monkeypatch, preserve the monkeypatch
  path with a compatibility export. Do not modify the test to chase the new
  private location.
- Golden/render tests protect byte-level output. Do not hand-edit expected
  artifacts when a generator should own them.
- Add regression tests before deleting compatibility shims or legacy barrels.
- Behavioral tests assert single invariants. If a test grows past ~30 LOC, split it.
- Tests that reveal production bugs are marked xfail with a reason and the bug
  is filed. Do not modify production code in a test-coverage commit.
```

### `ui/src/AGENTS.md` (frontend, if applicable)

```markdown
# ui/src/ Guardrails

- Do not edit ui/src/api/generated/ by hand. Update backend types and regenerate
  via the named generator.
- API endpoint methods stay explicit. Use shared transport helpers for path/query/
  idempotency, not a broad CRUD factory.
- Page components over 400 LOC need an architecture-test allowance and a split plan.
- Revision-aware mutations must preserve revision-header behavior and surface
  revision conflicts.
- UI behavioral tests assert what the user sees and does (visible text, accessible
  roles, click events). Do not assert on internal state, data-testid implementation
  details, or component class names.
```

### Per-integration `AGENTS.md` (if an integration has specific gotchas)

For external-system packages with non-obvious quirks. Only add when the rules don't fit at the `lib/providers/AGENTS.md` level.

```markdown
# lib/providers/<name>/ Guardrails

- The lifecycle files own canonical helpers:
  - registry.py owns registry record construction, contextual records, etc.
  - diff.py owns diff preview, pre-state capture, live-state verification.
  - apply.py owns execution and mutation result bookkeeping.
  - plan.py and artifacts.py are orchestration only; they import canonical
    helpers, they do not own them.
- Compatibility shims at the package root re-export only. They must not define behavior.
- [Any provider-specific quirk]
```

### Audit / cleanup directory `AGENTS.md`

For projects with an active cleanup roadmap.

```markdown
# docs/engineering/audits/ Guardrails

- Scorecard rows say "Complete" only when every acceptance criterion in the
  relevant plan is met with measurable evidence.
- "Partial" is preferred over "Complete" when any follow-up remains.
- Decision Gaps are filed here, not in chat history. When a refactor surfaces a
  design question, write it down with file:line citations and current behavior.
- Inventory baselines (clone_inventory_*.json or similar) are reference artifacts.
  Do not edit by hand; regenerate via the named tool.
```

## Anti-patterns in instructions-file content

Things that look like rules but don't function as guardrails:

### Vague aspirations

Bad: "Code should be clean and well-organized."
Good: "Files over 600 LOC require an architecture-test allowlist entry with a reason comment."

The first is a value statement; the second is a rule that fails when violated.

### Style preferences without enforcement

Bad: "Prefer composition over inheritance."
Good: "No universal BaseProvider class. Shared code is family-scoped or generic utility code."

The first is a heuristic that lets the agent decide; the second is a constraint the agent honors.

### Process descriptions

Bad: "When making a change, consider its impact on other modules."
Good: "Provider modules must not import publish domain internals."

The first is advice; the second is a checkable rule.

### Long explanations

Bad: A 3-paragraph essay on why migrations matter.
Good: "Existing migrations are immutable once committed. Add a new migration for schema changes."

The rule is the point. The why goes in an ADR if it matters.

### Rules that everyone already knows

Bad: "Write tests for new code."
Good: [omit; this isn't worth an instructions-file line]

Instructions files are for rules the agent would otherwise get wrong. Universal rules don't need restating.

## How to evolve instructions files

Add a rule when:

- A refactor produces a regression that a rule would have prevented.
- A pattern emerges that you want preserved across all future changes in that directory.
- An ADR is approved and needs local enforcement.

Remove a rule when:

- The constraint it encoded has been lifted by an architecture change.
- The rule is consistently honored without enforcement (it's become culture).
- The rule turns out to be wrong.

Keep a rule even if it seems obvious when:

- The agent has violated it in the past (memory is unreliable).
- The cost of restating is low (one bullet) and the cost of violation is high.

## How agents interpret instructions files

Patterns to know:

**Compliance is real but not perfect.** Agents honor explicit prohibitions ("Do not create misc.py") more reliably than implicit preferences ("Keep helper modules small"). Phrase rules as prohibitions where possible.

**Specificity beats generality.** A rule that cites a file path is honored more reliably than a rule that describes a pattern. "Do not edit ui/src/api/generated/" is honored more reliably than "Do not edit generated code."

**Order matters within a file.** Agents read top-down and weight earlier rules more heavily when rules conflict. Put the most load-bearing rules first.

**The file is read on every relevant action.** This is why instructions files outperform conversation memory. Conversation is forgotten; the file is re-read.

## Starting from zero

For a new project, install these on day one:

1. Root `AGENTS.md` — the cross-cutting rules.
2. `tests/AGENTS.md` — the test discipline.
3. One per major directory that will see refactoring.

Total: ~3–5 files at project start. Add more as patterns emerge.

Don't try to write all the directory files upfront. The rules that matter are the ones that emerge from real failures. Wait for the failure, then encode the rule.

## For high-stakes projects

When using the pattern in `HIGH_STAKES_PROJECT_PLAYBOOK.md`, the root `AGENTS.md` becomes the *constitution* layer of the three-layer model. It points at the project documents (roadmap, slice catalog, contract matrix), names the readiness tool, and states the irreversible-action rule. The root file template above already includes these hooks; for higher-stakes projects, also list:

- The path to the readiness tool and the command to invoke it.
- The slice closeout format (or a pointer to the catalog row that defines it).
- The exact list of irreversible actions that require human authorization.
- The freshness policy for evidence files.

Keep the root file under ~200 lines even in high-stakes projects. Anything longer than that goes into the project documents layer, not the constitution.
