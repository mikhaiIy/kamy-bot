# Pstack arena cross-judge

**Baseline commit:** `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`

## Scope and method

The two candidate artifacts were read end to end after parallel generation:

- `candidates/aggregate.md`: WorkflowRun aggregate and command boundary.
- `candidates/event-log.md`: append-only workflow event log with status projection.

Both cover the fixed first slice and remain design-only. The arena fan-out used independent OMP task agents and separate output paths. The attempted OMP cross-judge did not produce its requested artifact; it was stopped under the recovery instruction rather than waiting for or inventing a missing path. This report is the coordinator's direct cross-judge using the same rubric and full candidate contents. No production code, GitHub mutation, or worker launch occurred.

## Criterion scores

| Criterion | Aggregate candidate | Event-log candidate | Cross-judge evidence |
|---|---:|---:|---|
| Domain clarity | 5 | 4 | Aggregate makes `WorkflowRun` the durable unit and keeps the first slice's state chain compact. Event-log has precise facts but introduces streams, envelopes, commands, projections, and a control stream before they are required. |
| Interface depth and locality | 5 | 4 | Aggregate has three public operations and keeps sequencing/recovery behind `WorkflowService`. Event-log's `dispatch` is deep, but callers still construct and sequence `EnsureGitHubIssue`, `CheckOwnership`, and `StartDurableRun` from status. |
| Failure-case coverage | 5 | 5 | Both explicitly cover duplicate delivery, provider timeout after creation, durable write failure, restart, stop between transitions, idempotency, and stale controller versions. Event intent/result facts are particularly strong for recovery evidence. |
| Simplicity | 4 | 2 | Aggregate accepts one snapshot/CAS model and direct status for this slice. Event-log adds atomic batches, event schema evolution, reducer freshness, replay, command receipts, and a separate controller stream; these are defensible but mostly future-facing here. |
| Testability | 5 | 4 | Aggregate's service seam plus fake adapters directly exercise observable scenarios. Event-log has excellent replay fixtures, but projection lag and event schema behavior enlarge the test surface for a slice that only needs current status. |
| Agent executability | 5 | 4 | Aggregate exposes intent-level `admit`/`reconcile` and keeps workers behind the Scope gate. Event-log records the same gate but its reconciliation caller has to understand status-to-command progression. |
| Operational safety | 5 | 5 | Both make Project/controller identity, durable idempotency, provider idempotency, and two-gate autonomy explicit. Event-log's intent/result trace is a strong operational signal; aggregate's claim/CAS boundary is simpler to operate. |
| **Total** | **34/35** | **28/35** | Aggregate wins on locality and first-slice proportionality without losing the required safety properties. |

## Red-flag screen

### Aggregate candidate

- **Shallow module:** no. The small public surface hides claim ownership, external idempotency, transition validation, CAS retry, and recovery.
- **Information leakage:** no. GitHub and durable-store representations remain behind adapters.
- **Temporal decomposition:** avoided. Public calls express admission, reconciliation, and status rather than issue creation and ownership stages.
- **Pass-through methods:** no. The service owns the decision; adapter seams translate real external systems.

### Event-log candidate

- **Shallow module:** no for `dispatch`, but the command handler/module map is broad for this slice.
- **Information leakage:** mostly avoided. Domain events are clean, but event sequence/freshness becomes part of caller-visible status semantics.
- **Temporal decomposition:** partially present at caller level. The reconciliation example branches on projected issue and ownership states, then dispatches separate phase commands.
- **Pass-through methods:** no direct violation; the event handlers add idempotency and fact recording.

## Recommendation

Use the aggregate candidate as the base. It is the smallest deep boundary that supports the required first-slice behavior and future extension. Graft these event-log ideas into the base without adopting event-log-first persistence:

1. Represent every externally initiated operation with a stable operation/idempotency key and retain an observable evidence record.
2. Treat intent and external result as recovery facts inside the durable run record, so a crash between adapter call and transition is reconciled rather than guessed.
3. Expose durable revision/operation evidence and an explicit freshness/recovery status in verification, while keeping the public status view current and side-effect free.
4. Keep event/replay as a later internal persistence option, not a public first-slice contract.

The selected proposal therefore remains snapshot/CAS and aggregate-centric, with event-shaped evidence fields where they improve recovery. It does not average the two shapes or make callers coordinate their internal stages.
