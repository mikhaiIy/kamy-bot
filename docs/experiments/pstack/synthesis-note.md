# Pstack synthesis note

**Baseline commit:** `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`

## Decision

Select `candidates/aggregate.md` as the base and publish the refined design in `selected-proposal.md`. The aggregate/command shape scored 34/35 against the pstack rubric. The event-log/projection shape scored 28/35. The aggregate wins because the first slice needs current durable status and one admission decision, not a public replay system; it hides more sequencing behind a smaller surface and keeps state directly queryable.

## Arena record

Two independent OMP task agents produced structurally different packages at separate paths:

- `candidates/aggregate.md`: current `WorkflowRun` snapshot is the unit of ownership; `WorkflowService.admit`, `.reconcile`, and `.status` are the public operations; CAS and operation keys provide durable idempotency.
- `candidates/event-log.md`: ordered immutable intent/result facts are primary; an atomic append boundary and projection provide replay and explicit freshness.

Both candidates passed the basic red-flag screen. The event-log candidate's caller-level reconciliation example exposes more temporal command progression, and its event/schema/projection machinery is disproportionate for this slice. It remains a valuable alternative, not a discarded implementation detail.

The attempted OMP arena cross-judge did not write its requested path. Under the recovery instruction, it was stopped rather than awaited. The coordinator then completed the same criterion-by-criterion cross-judge after reading both artifacts end to end. The full score and evidence are in `arena-cross-judge.md`; the rubric is in `scoring-rubric.md`.

## Grafts from the losing candidate

The selected aggregate proposal incorporates these event-log ideas:

1. Stable operation/idempotency keys for every external operation.
2. Durable intent and external-result evidence that distinguishes unknown outcomes from committed results.
3. Atomic evidence/state/revision commits and replay-style failure scenarios.
4. Adapter traces and durable revision/operation records as operational evidence.

The public model remains snapshot/CAS and aggregate-centric. Event streams, public replay, projection freshness, and event-schema evolution are later options, not hidden requirements of callers.

## Interrogate result

The selected proposal was adversarially reviewed in three stable review lanes under the interrogate rubric. The review artifacts are `interrogate-reviewer-a.md`, `interrogate-reviewer-b.md`, `interrogate-reviewer-c.md`; the synthesized verdict is `interrogate-verdict.md`.

Consensus findings to act on before implementation:

- A and C: version lookup plus version partitions do not enforce a Project-level controller handoff. Add a monotonic fence epoch and stable cross-version WorkRequest claim, with explicit handoff race semantics.
- A and B: a named GitHub idempotency key is insufficient. Define deterministic lookup/create, unknown-outcome handling, and provider evidence.
- B: derive canonical intake identity and digest at the boundary; do not trust independently supplied identity/digest fields.
- A and B: complete the result/evidence algebra for identity collision, retryable failure, human recovery, and not-called/unknown-result/committed-result states.

Status consistency, Human decision provenance, and the later Worker extension seam remain considered risks. They are recorded without adding speculative later-phase types. The selected proposal includes these pre-implementation amendments and remains design-only.

## Rejected shapes

The public temporal pipeline, GitHub-as-source-of-truth, queue plus mutable status table, stateless controller with direct database procedures, global singleton registry, and speculative full-factory state machine were rejected. Reasons are recorded in `rejected-alternatives.md`; none is rejected merely for implementation style.

## Scope boundary

The pstack run designs only:

```text
Intake source → Work request → GitHub issue → ownership check
→ durable Workflow run → read-only status
```

It includes one Project profile, repository, and Intake source; duplicate intake; controller restart; a stop between transitions; and controller-version separation. It records the Scope and high-risk Delivery gates but does not approve either, start Workers, or design Discovery, Triage, implementation, review, risk, or merge. Learning remains a separate read-only workflow for proposals and Policy releases.

## Checkpoint

This is the requested stop-before-implementation checkpoint. No production code was changed. No GitHub issue was changed. No production worker was launched. The next implementation step is explicitly deferred until the four contract blockers from `unresolved-decisions.md` are resolved and the design is approved.
