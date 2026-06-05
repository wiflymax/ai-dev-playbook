# Readiness Tool Specification

The contract for the machine-checkable gate enforcer described in `HIGH_STAKES_PROJECT_PLAYBOOK.md`.

This spec is implementation-agnostic. Build the tool in whatever language fits your project — Python, Go, Rust, TypeScript, shell. What matters is the input/output contract and the refusal rules below.

## Purpose

The readiness tool answers one question:

> Given everything that is supposed to be true before this project can ship, is it actually true right now?

The tool exists because agents cannot be trusted to grade their own homework, and humans cannot manually verify every claim across multi-week multi-session work. The tool is the third party. Its verdict is authoritative.

The tool must be:

- **Cheap to run.** Sub-second on local dev, sub-minute in CI.
- **Deterministic.** Same inputs produce same output.
- **Read-only.** It inspects evidence; it never produces it.
- **Self-contained.** No network calls to make a verdict. Reads files, exits.
- **Pessimistic by default.** When in doubt, block.

## What the tool reads

The tool reads three kinds of inputs:

### 1. Closeout reports

Markdown files at known paths (e.g., `docs/<project>/reports/PN_*.md`). Each report ends with a binding line:

```
Ready to advance: yes
```

or

```
Ready to advance: no
```

The tool parses this line from every required report. Anything else (`yes, with caveats`, `partial`, no line at all) is treated as `no`.

### 2. Evidence files

JSON files at known paths (e.g., `artifacts/<project>/<gate-name>.json`). Each evidence file represents one proven claim. The tool verifies:

- The file exists.
- The file is non-empty.
- The file matches the declared schema for its gate type.
- The file's content satisfies the gate's refusal rules (see below).

### 3. Catalog and policy documents

The slice catalog (`02_SLICE_CATALOG.md`) and the change-sync/policy document (`05_CHANGE_SYNC_AND_POLICY.md`) declare what's required. The tool consults them to know:

- Which closeout reports must exist.
- Which evidence files must be present.
- Which slices are explicitly blocked vs. silently missing.

### 4. Inventory equivalence outputs

The inventory-and-equivalence document (`06_INVENTORY_AND_EQUIVALENCE_GATES.md`) declares, for each surface the old system exposes, an inventory file and an equivalence tool. The readiness tool consults both:

- The inventory file (what the old system has, e.g. `route_inventory.json`).
- The equivalence tool's output (the diff between old and new, with `missing` and `extra` lists).

The tool refuses to clear unless every declared inventory has a current equivalence output and every entry in the `missing` list either has a matching slice in flight or a retirement amendment on file. See "Inventory-equivalence refusals" below.

## What the tool emits

A single JSON document on stdout (or written to `--json-out <path>`) with this shape:

```json
{
  "status": "blocked",
  "generated_at": "2026-06-03T18:42:00Z",
  "tool_version": "1.0.0",
  "blockers": [
    {
      "code": "phase_not_ready",
      "report": "P7_INITIAL_APPLY_READINESS_REPORT.md",
      "message": "P7_INITIAL_APPLY_READINESS_REPORT.md says Ready to advance: no"
    },
    {
      "code": "evidence_missing",
      "gate": "lab_dry_run_apply",
      "subsystem": "dnac",
      "expected_path": "artifacts/lab/dnac/lab-dry-run-apply.json",
      "message": "DNAC is missing required evidence: lab_dry_run_apply"
    },
    {
      "code": "evidence_invalid_source",
      "gate": "production_mutation_jobs_zero",
      "expected_source": "postgres",
      "actual_source": "sqlite",
      "message": "Production mutation-job evidence must come from Postgres, not SQLite"
    }
  ],
  "phase_reports": [
    {
      "report": "P5_CLOSEOUT_AUDIT_REPORT.md",
      "exists": true,
      "ready_to_advance": "yes"
    }
  ],
  "required_reports": [
    "P0_INITIAL_CONTRACT_TOOLING_REPORT.md",
    "P1_RUST_WORKSPACE_SKELETON_REPORT.md"
  ]
}
```

Field semantics:

| Field | Meaning |
|---|---|
| `status` | One of `green`, `blocked`. `green` only if `blockers` is empty. |
| `generated_at` | ISO 8601 UTC timestamp. |
| `tool_version` | Semantic version of the tool itself. Bump when the schema or rules change. |
| `blockers` | Array of every reason the project cannot ship right now. |
| `phase_reports` | Inventory of each required closeout report and its current status. |
| `required_reports` | Declared list of reports that must exist. |

Exit codes:

- `0` if `status == "green"`.
- Non-zero (e.g., `1`) if `status == "blocked"`.
- Non-zero (e.g., `2`) on tool-internal errors (missing config, malformed input). Distinguishable from blocker exits.

## Blocker code conventions

Every blocker has a `code` field. Codes are stable strings (lowercase, underscores) that future agents can match against. New codes can be added; existing codes never change meaning.

Recommended starter codes:

| Code | Meaning |
|---|---|
| `phase_not_ready` | A required closeout report says `Ready to advance: no` or is missing the line. |
| `report_missing` | A required closeout report file does not exist. |
| `evidence_missing` | An expected evidence file does not exist at its declared path. |
| `evidence_empty` | An evidence file exists but is empty or contains only placeholder content. |
| `evidence_invalid_schema` | An evidence file exists but doesn't match its declared schema. |
| `evidence_invalid_source` | An evidence file's content reveals it came from a disallowed source (e.g., SQLite where Postgres is required, fake-client where lab is required). |
| `evidence_stale` | An evidence file exists but is older than the freshness window allowed by policy. |
| `subsystem_not_ready` | A per-subsystem readiness record (e.g., per-provider, per-region) declares missing artifacts. |
| `gate_not_green` | A named gate in the contract/test matrix is not satisfied. |
| `irreversible_action_unauthorized` | An evidence file describes an irreversible action without the required human-authorization markers. |
| `slice_floor_unmet` | A slice claims complete but its closeout shows numerical floor not reached without a stop condition cited. |
| `inventory_mismatch_missing` | The old system declares a surface (route, operation, scheduled job, entity, provider) that the new system does not expose, and no retirement amendment exists. |
| `inventory_mismatch_extra` | The new system exposes a surface the old system never had, without an addition amendment naming why. |
| `inventory_not_diffed` | An inventory file was captured but no equivalence diff against the target system has been performed (or recorded) for this gate cycle. |

Each blocker should carry enough context (file path, gate name, subsystem ID) that an agent reading the JSON can take action without guessing.

## Refusal rules

The tool must refuse evidence under these conditions. The list is not exhaustive — add project-specific rules as the project requires.

### Universal refusals

- Empty files. An empty `artifacts/.../foo.json` is not evidence.
- Files containing only placeholder content (`TODO`, `TBD`, `# placeholder`, `null`, `{}`). The tool defines what placeholder means via a small allowlist of forbidden substrings.
- Files at wrong paths. If the catalog declares `artifacts/lab/dnac/lab-dry-run-apply.json`, evidence at `artifacts/dnac-lab.json` does not count.
- Files older than the freshness window declared by the change-sync policy.
- Closeout reports without a parseable `Ready to advance:` line, or with `no` / `partial` / `with caveats`.

### Source-based refusals

For evidence that has a required source (database, environment, tool):

- Evidence that declares an actual source different from its required source is rejected with `evidence_invalid_source`.
- Evidence with no source declaration where one is required is rejected with `evidence_missing` (it's not real evidence, it's an assertion).

Example: production mutation-job-zero evidence is required to come from a Postgres query. Evidence with `"source": "sqlite"` is rejected.

### Authorization-based refusals

For irreversible-action evidence:

- Must include explicit human-authorization markers (named field with a non-empty value: `confirmed_by`, `approval_reference`, `safety_yes: true`, etc. — whatever the project's policy defines).
- Must include a scope record (which environment, which subsystem, which window).
- Must include rollback readiness reference.

If any of those is missing, refuse with `irreversible_action_unauthorized`.

### Schema-based refusals

For evidence with a declared schema:

- Must validate against the schema.
- Schema mismatches produce `evidence_invalid_schema` with the specific field that failed.

Use a real schema validator. Don't approximate with string matching.

### Inventory-equivalence refusals

For every surface inventory the project declares (typically: routes, operations, scheduled jobs, message handlers, entities, providers):

- The tool must consult the equivalence output that diffs the old-system inventory against the new-system surface.
- A surface in the old inventory that is not in the new system and not on the retirement-amendment list produces `inventory_mismatch_missing`. The blocker must name the surface (method+path for a route, handler name for a job, etc.), the owning phase/slice that would implement it, and the path to the inventory file that declared it.
- A surface in the new system that is not in the old inventory and not on the addition-amendment list produces `inventory_mismatch_extra`. Less common but worth catching — accidental new surfaces are how an agent quietly expands scope.
- An inventory file that exists but has no recorded equivalence output produces `inventory_not_diffed`. The principle: **a captured inventory that no tool diffs is decoration, and decoration must not let the readiness tool clear.** Inventories without paired equivalence tools are themselves blockers.
- Equivalence outputs older than the freshness window declared by the change-sync policy produce `evidence_stale`. When the old system changes a surface, the equivalence diff must rerun.

The principle here matters: **the readiness tool's job for inventories is to gate completeness, not just correctness.** A new system can satisfy every named gate and still be silently incomplete if no tool compares it against what the old system actually had. The inventory-equivalence refusals close that gap.

## Example input

A minimal evidence file the tool would accept (`artifacts/lab/dnac/lab-dry-run-apply.json`):

```json
{
  "schema_version": "1.0.0",
  "subsystem": "dnac",
  "action": "lab_dry_run_apply",
  "executed_at": "2026-06-03T14:22:00Z",
  "source": {
    "environment": "lab-east-1",
    "credentials_reference": "vault://dnac/lab-east-1",
    "isolated": true
  },
  "result": {
    "status": "success",
    "apply_result_artifact": "artifacts/lab/dnac/apply-result-2026-06-03.json",
    "post_state_artifact": "artifacts/lab/dnac/post-state-2026-06-03.json",
    "compliance_artifact": "artifacts/lab/dnac/compliance-2026-06-03.json"
  },
  "rollback_readiness": {
    "procedure_reference": "docs/.../rollback/dnac-lab.md",
    "rehearsed": true
  }
}
```

A closeout report the tool would accept (`docs/.../reports/P5_CLOSEOUT_AUDIT_REPORT.md`):

```markdown
# Phase 5 Closeout Audit

Date: 2026-05-31

## Result

All P5 slices closed with byte-parity evidence.

## Evidence

- artifacts/parity/P5-canonical-snapshot-parity.json
- artifacts/parity/P5-workspace-document-parity.json
- [other paths]

Ready to advance: yes
```

## Example output

A blocked status:

```json
{
  "status": "blocked",
  "generated_at": "2026-06-03T18:42:00Z",
  "tool_version": "1.0.0",
  "blockers": [
    {
      "code": "subsystem_not_ready",
      "subsystem": "dnac",
      "message": "DNAC is missing required evidence: lab_dry_run_apply, production_canary"
    },
    {
      "code": "irreversible_action_unauthorized",
      "evidence": "artifacts/canary/aruba-mc/production-canary.json",
      "missing_fields": ["confirmed_by", "rollback_readiness.procedure_reference"],
      "message": "Aruba MC canary evidence lacks human authorization markers"
    }
  ],
  "phase_reports": [
    {"report": "P0_REPORT.md", "exists": true, "ready_to_advance": "yes"},
    {"report": "P5_REPORT.md", "exists": true, "ready_to_advance": "yes"},
    {"report": "P7_REPORT.md", "exists": true, "ready_to_advance": "no"}
  ],
  "required_reports": ["P0_REPORT.md", "P1_REPORT.md", "P5_REPORT.md", "P7_REPORT.md"]
}
```

A green status:

```json
{
  "status": "green",
  "generated_at": "2026-06-04T09:00:00Z",
  "tool_version": "1.0.0",
  "blockers": [],
  "phase_reports": [
    {"report": "P0_REPORT.md", "exists": true, "ready_to_advance": "yes"},
    {"report": "P1_REPORT.md", "exists": true, "ready_to_advance": "yes"},
    {"report": "P5_REPORT.md", "exists": true, "ready_to_advance": "yes"},
    {"report": "P7_REPORT.md", "exists": true, "ready_to_advance": "yes"}
  ],
  "required_reports": ["P0_REPORT.md", "P1_REPORT.md", "P5_REPORT.md", "P7_REPORT.md"]
}
```

When `status == "green"`, the project has cleared every gate the tool knows about. It does not mean the project is automatically safe to ship — it means nothing the tool can verify is blocked. Humans still own the final go decision.

## Integration points

### Pre-commit hook

Run the tool before any commit on the project branch. Fail the commit if status is blocked *and* the commit touches files that should be gate-clean (closeout reports, evidence files, slice catalog).

Don't fail every commit on every file change. The hook should know which commits matter.

### CI

Run the tool on every PR. Post the blocker list as a PR comment. Fail the build only when a PR claims to close a gate that the tool says is not closed.

### Agent workflow

Agents must run the tool before claiming a phase or project is ready. The closeout report template should include a verification line:

```
Verification:
- Readiness tool output: artifacts/readiness-<date>.json (status: <green|blocked>)
```

If an agent's closeout claims `Ready to advance: yes` but the readiness tool output saved alongside it shows `blocked`, the slice fails review automatically.

### Cutover gate

For migration-style projects, the tool's `status == "green"` is a *necessary* condition for cutover. Not sufficient — human approval, calendar coordination, and runtime rehearsal remain required — but the cutover does not proceed if the tool is blocked.

## Anti-patterns the tool must avoid

Things a readiness tool must never do:

### Self-reported pass

The tool must never accept "the agent says it passed" as evidence. Every passing claim has a file. The tool reads the file. If the file says pass and the schema validates and the source is right and the freshness is current, the gate is green. Otherwise it is not.

### Default-to-green

When the tool encounters an unrecognized state, the default is `blocked`, not `green`. New evidence types, new gates, new subsystems — all default to blocked until explicitly declared and validated. "Probably fine" is not a verdict the tool is allowed to render.

### Warning instead of blocking

The tool emits blockers, not warnings. There is no middle tier. A "warning" the project can ship despite is a gate that doesn't work. Either the condition matters (block) or it doesn't (don't emit).

### Optional checks

Every check in the tool is mandatory. Optional checks accumulate into "we always skip those" and the gate erodes. If a check is not worth running every time, remove it.

### Embedded business logic

The tool checks gates; it does not enforce business policy. Whether a particular canary scope is acceptable for a particular vendor is a human decision recorded in evidence. The tool verifies the evidence has the required shape and authorization markers; it does not opine on whether the scope is wise.

### Hidden state

The tool reads files. It does not consult a database, a remote API, or a cached prior run. Inputs are visible in the repo. Outputs are written to a known path. State is on disk, not in the tool.

## Evolution

The tool's rules will get stricter over time. That is correct.

When to add a new refusal rule:
- A reviewer (human or agent) identifies a gap where ambiguous evidence could pass.
- A real incident reveals an evidence type that should have been refused.
- A new gate is added to the contract/test matrix.

When to change an existing rule:
- The rule turned out to refuse legitimate evidence. Tighten the rule's specificity rather than relaxing it.

When to remove a rule:
- The underlying gate has been removed from the project.
- Almost never otherwise.

Each rule change bumps `tool_version`. Evidence written under an older version may need regeneration. The tool reports the version it expects vs. the version the evidence declares.

## What the tool is not

- It is not a test runner. Tests are evidence; the tool reads their reports.
- It is not a CI system. It exits with a status; CI decides what to do with that status.
- It is not a project tracker. It reports on gate status, not feature progress.
- It is not a documentation generator. It validates documents; it does not produce them.
- It is not a substitute for human review. It is the prerequisite for human review.

## Summary

Input: files at known paths (closeout reports, evidence JSON, catalog declarations).
Output: a single JSON document with a `status` and a `blockers` array.
Behavior: pessimistic, deterministic, read-only, self-contained.
Refusals: empty files, placeholders, wrong paths, wrong sources, missing authorization, schema mismatches, stale evidence.
Exit codes: 0 green, non-zero blocked, distinguishable from tool errors.
Integration: pre-commit, CI, agent workflow, cutover gate.
Authority: when the tool says blocked, the project is blocked. No exceptions.

The tool's job is to be the third party that can't be argued with. If you build it well, the rest of the playbook works. If you skip it or weaken it, the rest of the playbook is theater.
