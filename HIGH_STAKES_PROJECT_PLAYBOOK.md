# High-Stakes Project Playbook

The enforcement-grade pattern for AI-driven work on systems where being wrong is expensive.

This document is vendor-neutral. It applies to any coding agent (Claude Code, Codex, Cursor, Aider, custom harnesses) and any tech stack. For a worked example, see `docs/engineering/rust-migration/` in this repository.

## When to use this playbook

Reach for this playbook when at least two of these are true:

- The work spans multiple weeks and multiple agent sessions.
- A wrong outcome would produce a customer-visible incident, data loss, or a hard-to-roll-back state change.
- The work involves external systems whose state cannot be cleanly undone.
- The work is mostly agent-driven and a human cannot review every line.
- The work runs in parallel with continued evolution of an existing system.

For lighter work (single-feature refactors, coverage expansions, one-shot scripts) use the pattern in `PRINCIPLES.md` and `PLAN_AND_PROMPT_TEMPLATES.md`. This playbook is heavier; the overhead pays back only at scale.

## The three-layer model

Every high-stakes agent-driven project needs three distinct layers. Conflating them is the most common failure mode.

| Layer | What it does | Where it lives |
|---|---|---|
| **Agent configuration** | Defines what the agent is allowed to do and what it always reads | Agent instructions file at repo root, settings, hooks, tool allowlists |
| **Project documents** | Defines what the work is, what the contracts are, what counts as done | Markdown in the repo, loaded on demand |
| **Readiness tool** | Machine-checks whether claimed progress is real | Script that emits structured JSON, exits non-zero if anything is blocked |

If you build only the documents, agents drift because nothing forces them to read the rules every turn. If you build only the agent config, agents have no detailed plan to execute. If you build both but skip the readiness tool, agents grade their own homework. **All three are required.**

See `AGENT_INSTRUCTIONS_STARTER_PACK.md` for the agent configuration layer. See `READINESS_TOOL_SPEC.md` for the readiness tool layer. The rest of this document is about the project documents layer.

## The project document set

Eight documents form the skeleton. Names and exact structure can vary; the *roles* are non-negotiable.

| Document | Role | Loaded by |
|---|---|---|
| `00_FEASIBILITY_STATEMENT.md` | Why this exists. Risk surface. Non-negotiable constraints. Verdict. | Agents at first orientation |
| `01_ROADMAP.md` | Phase ladder, risk-last. Each phase has an exit gate stated in evidence terms. | Agents starting any phase |
| `02_SLICE_CATALOG.md` | Roadmap broken into commit-revertible units with stable IDs. | Agents picking up a slice |
| `03_METHOD_OF_PROCEDURE.md` | The standard procedure every slice follows. | Agents executing any slice |
| `04_CONTRACT_AND_TEST_MATRIX.md` | What "preserve behavior" means, surface by surface, with test anchors. | Agents touching a contract surface |
| `05_CHANGE_SYNC_AND_POLICY.md` | What stays stable, what's frozen, what requires approval. AI guardrails. | Agents making changes that touch the base system |
| `phases/PHASE_N_*.md` | Per-phase executable brief. Includes numerical floors and agent executor brief. | Agents executing that phase |
| `development-guide/PRINCIPLES.md` | How to think about the work for *this* project specifically. | Agents at first orientation |

The roadmap and slice catalog are the most-read documents. Keep them tight and cross-referenced. Long roadmaps don't get read.

Skeleton headings for each of the eight documents are in `PLAN_AND_PROMPT_TEMPLATES.md` under "High-stakes project document set."

## The slice — the atomic unit of work

Every unit of work is a **slice**. Slices have:

- A stable ID (e.g. `P3-S2` = Phase 3, Slice 2). IDs never get reused.
- A name describing the deliverable in evidence terms.
- Primary verification command(s).
- Stop conditions ("if X, halt and reassess").
- Required evidence files.
- A binding closeout format.

The closeout format is identical for every slice. No variation:

```
Slice: P3-S2
Roadmap phase:
Phase plan:
Files changed:
Tests added/updated:
Verification output:
Evidence files:
Bugs discovered:
Ready to advance: yes/no
```

If `Ready to advance: no`, the agent **stops**. Doesn't move to the next slice. Doesn't argue. The next session picks up from the no.

## Numerical floors are binding

Every slice has a floor table. The floor is what the slice must hit to close:

```
| Slice | Minimum floor |
|---|---:|
| P7-S1 | Every provider covers auth success/failure, retry, timeout, 429, 5xx, malformed response, and redaction. |
| P7-S4 | Each provider has fake-client matrices or is blocked from cutover ownership. |
```

The floor table answers "is this done?" Not prose. Not vibes. Not "looks good to me."

Two rules:
1. **Floors are per-slice, not just per-phase.** A phase-level floor is gameable across slices.
2. **A slice may close at ≥90% of its floor only when a documented stop condition blocks the remaining gap.** Below 90%, the slice fails and reopens.

## The readiness tool

The single highest-leverage artifact in the playbook. Without it, agents drift no matter how good the documents are.

The full contract is in `READINESS_TOOL_SPEC.md`. Build it **early** — before the first phase ships. It's the only thing standing between you and an agent that closes its own gates.

## The evidence-over-assertion rule

Every claim of progress is a **file with a verifiable shape**, not a sentence in a report. Examples:

| Claim | Evidence form |
|---|---|
| "Portable test suite passes" | JSON report at a stable path, generated by a specific tool, with a verifiable schema |
| "Lab apply succeeded against the external system" | Evidence bundle: apply-result artifact, post-state inspection, compliance check, rollback proof |
| "Subsystem X is ready for cutover" | Per-subsystem readiness JSON listing each required artifact and its file path |
| "Cutover rehearsal completed" | Backup reference, restored environment reference, smoke output, rollback startup output |

If a claim cannot be reduced to a file at a stable path, it cannot be a gate.

## The irreversible-action rule

Some actions must **never** be closed by the agent on its own confidence:

- Production mutation against external systems.
- Apply or write against a real lab environment.
- Schema migrations.
- Force pushes, branch deletions, tag deletions.
- Customer-visible deploys.
- Sending external messages (chat, email, public PR comments).

For these, the agent's role is to **prepare** evidence. The human's role is to **authorize** the action. The readiness tool's role is to **refuse** evidence that doesn't show explicit human authorization (a signed-off flag, an approval reference, a recorded confirmation header).

This is the rule that prevents agent velocity from outpacing operational safety.

## The freeze doctrine

At some point in a parallel rewrite or extended migration, the base system must stop accepting feature work in the surfaces being migrated. Otherwise, the parity evidence the new system depends on rots faster than it can be refreshed.

Write the freeze policy **before you need it**. Define:

- Which phase or date triggers the freeze.
- Which surfaces are frozen (usually: writes, external-system integrations, durable job runtime — not reads, auth, presentation).
- What's allowed during freeze (security fixes, production bug fixes).
- What's not allowed (new features, new contract behavior).
- What every allowed post-freeze change must update (fixtures, parity expectations, evidence reports).

If you write the freeze policy under pressure, it'll be too late.

## Outside review adds gates, not just commentary

Periodic reviews by a fresh agent or a fresh human read are valuable. The discipline: review findings should land as **new gates the executor then has to satisfy**, not as commentary in chat.

Pattern:
1. Reviewer reads the project documents cold.
2. Reviewer names risks or gaps.
3. Each gap that survives discussion becomes a new gate in the contract/test matrix, a new floor in a phase plan, or a new check in the readiness tool.
4. The executor's next session has to clear the new gate.

This is how the documents get *stricter* over time, not looser.

## Agent configuration that supports the playbook

The repo-root agent instructions file is the constitution. It should:

- Point at the required reading order.
- State the hard rules (readiness tool is authoritative, never edit goldens by hand, never close irreversible-action gates, etc.).
- Reference the slice closeout format.
- Name the tools.
- Say "when in doubt, ask."

See `AGENT_INSTRUCTIONS_STARTER_PACK.md` for the actual shape and per-vendor file conventions (`AGENTS.md`, `CLAUDE.md`, `.cursorrules`, etc.).

Settings, hooks, and tool allowlists complement the constitution:

- **Tool allowlists**: name the safe tools, deny destructive defaults.
- **Pre-commit / commit-msg hooks**: refuse commits without a slice ID, run the readiness tool, refuse hook-bypassing flags.
- **Subagent definitions** (where the harness supports them): specialized roles (parity reviewer, safety reviewer) with narrow tool access.

## What this playbook prevents

The five failure modes that kill most agent-driven rewrites:

1. **Self-graded completion.** Agent claims done; nobody can verify. → Prevented by readiness tool.
2. **Drift treadmill.** Base system evolves; new system's evidence rots; never converges. → Prevented by freeze doctrine and change-sync policy.
3. **Hidden product changes.** Agent "improves" behavior the project depended on. → Prevented by contract/test matrix.
4. **Irreversible-action mishaps.** Agent runs a production write on its own judgment. → Prevented by irreversible-action rule.
5. **Lost context across sessions.** New session re-litigates settled decisions. → Prevented by ADRs, change-sync log, and slice catalog.

Skipping any of the three layers (agent config / project documents / readiness tool) reintroduces the failure mode that layer prevents.

## What "done" looks like

A high-stakes project under this playbook is done when:

- Every slice in the catalog is closed or explicitly blocked with a roadmap amendment.
- Every phase plan has `Ready to advance: yes`.
- The readiness tool exits zero.
- The full contract/test matrix passes.
- For migrations: rehearsal on a restored production-like environment has passed, and rollback has been rehearsed.
- For external-system work: lab evidence and approved narrow production canary evidence is on file for every affected target.

If any of the above is missing, the project is not done. The readiness tool will say so. Listen to it.

## Recommended bootstrap order for a new high-stakes project

1. **Write `00_FEASIBILITY_STATEMENT.md` first.** What you're protecting, what could go wrong, what the verdict is.
2. **Write `01_ROADMAP.md`.** Phases ordered risk-last. Each phase has an exit gate in evidence terms.
3. **Write `04_CONTRACT_AND_TEST_MATRIX.md`.** What "preserve behavior" means, surface by surface.
4. **Write `05_CHANGE_SYNC_AND_POLICY.md`.** Including the freeze doctrine and AI guardrails.
5. **Write `02_SLICE_CATALOG.md`.** Phase 0 and Phase 1 slices to start. Add more as phases approach.
6. **Build the readiness tool.** Even with only the early phases defined. Emit JSON. Refuse anything not proven.
7. **Write the root agent instructions file.** Constitution. Points at the above. States the hard rules.
8. **Write `phases/PHASE_0_*.md`.** Executable brief for the first phase, with numerical floors and agent executor brief.
9. **Execute.**

Steps 1–7 take a few days of focused work. They feel disproportionately heavy at the start. They become the load-bearing infrastructure by week three. Skipping any of them produces work that has to be redone under pressure later.

## A worked example

Everything in this playbook is in production use in `docs/engineering/rust-migration/` in this repository. That directory is the canonical reference implementation:

- `README.md` — entry doc
- `00_FEASIBILITY_STATEMENT.md` through `06_SLICE_CATALOG.md` — the document set
- `phases/PHASE_*.md` — per-phase plans with numerical floors and agent executor briefs
- `development-guide/` — project-specific overlay on the general principles
- `tools/check_rust_cutover_readiness.py` (at repo root) — the readiness tool
- `artifacts/rust-parity/*.json` — evidence files in their canonical shape

Read those files as a reference until something better replaces them. The patterns described above were proven there.

## Summary

Three layers (agent config, project documents, readiness tool). Eight documents in the project document set. Every slice closes with a binding format and a `Ready to advance: yes/no` gate. Numerical floors are binding. Evidence is files, not sentences. Irreversible actions need human authorization. Freeze early. Outside review adds gates. Build the readiness tool first.

If you do these things, the agent can do the labor and the result will be safe to ship.

If you skip any of them, the agent will produce work that looks done and isn't.

The discipline is the deliverable.
