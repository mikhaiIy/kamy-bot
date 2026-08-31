# Experiment 001 comparison

## Scope

This document compares the Matt and pstack development workflows for the first Kamy-Bot Agentic Development Workflow foundation slice.

The fixed slice was:

```text
Intake source
→ Work request
→ GitHub issue
→ ownership check
→ durable Workflow run
→ read-only status
```

Both sessions were design-only. Neither session changed production code, GitHub, or production workers.

## Run evidence

| Run | Herdr pane | Starting commit | Output directory |
| --- | --- | --- | --- |
| Matt | `w8:p5` | `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f` | `docs/experiments/mattp/` |
| pstack | `w8:p4` | `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f` | `docs/experiments/pstack/` |

Both runs used the same shared context and experiment brief. Matt did not read the pstack directory. Pstack did not read the Matt directory.

Both sessions used the same main working directory instead of separate Git worktrees. The output directories were separate, and no shared source files changed. This is acceptable for this design-only run, but future comparisons should use separate worktrees to enforce isolation.

The automatic clarification path was not exercised. Neither session needed a blocking question. Both sessions recorded assumptions and unresolved decisions.

## Artifact completeness

### Matt

The Matt run produced:

- `questions-assumptions.md`
- `scoring-rubric.md`
- `candidate-state-transition.md`
- `candidate-event-log.md`
- `candidate-evaluation.md`
- `selected-proposal.md`
- `synthesis-note.md`

The run used `grill-with-docs`, `domain-modeling`, and `codebase-design`. It produced two structural candidates, selected one, recorded rejected alternatives, and stopped before implementation.

### pstack

The pstack run produced:

- `candidates/aggregate.md`
- `candidates/event-log.md`
- `scoring-rubric.md`
- `arena-cross-judge.md`
- `selected-proposal.md`
- `interrogate-reviewer-a.md`
- `interrogate-reviewer-b.md`
- `interrogate-reviewer-c.md`
- `interrogate-verdict.md`
- `rejected-alternatives.md`
- `unresolved-decisions.md`
- `synthesis-note.md`
- `verification-evidence.md`

The first arena cross-judge task used a missing output path. The coordinator stopped that task and instructed the pstack session to recover from the actual candidate files. The pstack session completed the cross-judge, review, synthesis, and verification artifacts after recovery.

## Design convergence

Both workflows selected the same broad architecture:

- A transactional current Workflow run is authoritative.
- Immutable transition and evidence records support audit and replay fixtures.
- External provider calls use durable intent and reconciliation.
- Duplicate intake uses stable identity and idempotency.
- Project and Controller version scope every mutation.
- Controller-version handoff requires fencing.
- Read-only status performs no mutation or external call.
- Later Worker execution remains outside the first slice.
- The Learning workflow remains separate from the main Workflow.
- No existing Agentic framework becomes mandatory because of familiarity.

This convergence is strong evidence that the context and fixed task were understood correctly.

## Meaningful differences

### Matt strengths

- `grill-with-docs` made the product context and domain language explicit.
- The questions and assumptions artifact separates context-grounded answers from unresolved product decisions.
- The selected proposal defines a small `submit` and `read_status` interface.
- The proposal gives a detailed list of decisions that block production implementation.
- The workflow completed without a path-recovery failure.

Matt did not ask blocking questions because the revised context answered the first-slice and autonomy questions. This confirms that a strong context can reduce the need for a long interview.

### Pstack strengths

- `architect` and `arena` forced two complete structural candidates.
- The selected proposal gives a concrete `admit`, `reconcile`, and `status` interface.
- The state model explicitly names recovery states and gate facts.
- `interrogate` found concrete pre-implementation blockers:
  - Project-level Controller fencing.
  - Stable cross-version Work request claims.
  - GitHub lookup/create idempotency and unknown outcomes.
  - Canonical intake identity and digest derivation.
- Pstack recorded a stronger operational review trail.

### Pstack weaknesses in this run

- The first cross-judge task used a path that did not exist.
- The session required coordinator recovery before it could finish.
- The pstack artifacts are more detailed and require more reading.
- Some pstack instructions assume Cursor-specific execution concepts. The OMP prompt had to override those assumptions.

## Coordinator scoring

These scores use one common rubric. They are coordinator scores, not the scores produced by either workflow.

| Criterion | Matt | pstack | Evidence |
| --- | ---: | ---: | --- |
| Product understanding | 5 | 5 | Both preserved the full factory context and limited implementation to the first slice. |
| Domain clarity | 5 | 5 | Both defined Project, Work request, Workflow run, gates, evidence, and Controller version. |
| Interface depth and locality | 5 | 5 | Both hide sequencing, idempotency, reconciliation, and fencing behind a small Module Interface. |
| Failure-case coverage | 5 | 5 | Both cover duplicate intake, restart, partial writes, external ambiguity, and version separation. |
| Simplicity | 4 | 4 | Both avoid mandatory event sourcing, but both include substantial safety machinery. |
| Agent executability | 4 | 5 | Pstack gives the clearest concrete command and state protocol. Matt gives the clearest command vocabulary and constraints. |
| Verification quality | 5 | 5 | Both define observable interface-level scenarios and state that the evidence is planned, not executed. |
| **Total** | **33/35** | **34/35** | The difference is small. Both designs are viable. |

## Process scoring

| Criterion | Matt | pstack | Evidence |
| --- | ---: | ---: | --- |
| Alignment and terminology | 5 | 4 | Matt's domain process was stronger. Pstack understood the context but needed path recovery. |
| Candidate diversity | 4 | 5 | Both produced two candidates. Pstack used a clearer explicit arena process. |
| Decision record | 5 | 5 | Both recorded alternatives, rationale, unresolved decisions, and a selected design. |
| Adversarial pressure | 4 | 5 | Pstack ran three interrogate review lanes. Matt performed candidate evaluation but not a separate adversarial review. |
| OMP portability | 5 | 3 | Matt used the OMP environment directly. Pstack needed explicit replacement of Cursor-specific assumptions and coordinator recovery. |
| **Total** | **23/25** | **22/25** | Matt is the better default OMP workflow. Pstack adds stronger architecture pressure. |

## Recommendation

Use a composed workflow for the next Kamy-Bot task:

1. Use Matt `grill-with-docs`, `domain-modeling`, and `codebase-design` to settle product language, scope, and unresolved decisions.
2. Use pstack `architect` and `arena` when the architecture has a real structural fork.
3. Use pstack `interrogate` before implementation when the change affects concurrency, external effects, authorization, or recovery.
4. Use Matt `to-spec` and `to-tickets` after the design is approved.
5. Use one OMP implementation and verification path. Do not run two top-level workflow routers together.

Port the pstack processes, not its Cursor-specific runtime assumptions. The most valuable ports are:

- Caller-first architecture sketches.
- Two structurally different candidates.
- Criterion-based cross-judging.
- Explicit rejected alternatives.
- Adversarial review before implementation.
- Durable verification evidence.

Do not port pstack's Cursor model configuration, Cursor cloud-agent assumptions, Graphite workflow, or `gt` commands.

## Decision status

The comparison is complete. The architecture is not approved for implementation yet.

The next design step is to resolve the shared contract blockers recorded in both selected proposals, especially external idempotency, Controller-version fencing, ownership authority, status redaction, and Learning proposal approval.

No production implementation starts until those decisions are approved.
