# Matt synthesis note: Experiment 001

- Starting commit: `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`
- Run scope: Matt side only; design comparison stopped before implementation.
- Canonical shared inputs were not modified.
- No GitHub mutation, production worker launch, or production code change occurred.

## Artifacts

- `docs/experiments/mattp/questions-assumptions.md`
- `docs/experiments/mattp/scoring-rubric.md`
- `docs/experiments/mattp/candidate-state-transition.md`
- `docs/experiments/mattp/candidate-event-log.md`
- `docs/experiments/mattp/candidate-evaluation.md`
- `docs/experiments/mattp/selected-proposal.md`
- `docs/experiments/mattp/synthesis-note.md`

Every Matt artifact records the exact starting commit above.

## Fixed-slice result

The Matt run kept the scope at:

```text
Intake source → Work request → GitHub issue → ownership check → durable Workflow run → read-only status
```

It treats Discovery, Triage, Scope approval details, Worker implementation, review, risk checks, Delivery approval, merge, and Learning policy release as later capabilities. The two-gate autonomy model is preserved: ownership/authority first; Approved scope before later Worker execution. Learning remains a separate workflow.

## Candidates considered

### 1. Transactional state-transition engine — selected

Durable current run snapshot is authoritative. Immutable transition/evidence records preserve audit and replay-fixture value. A deep `Workflow engine` Module exposes `submit(command, idempotency_key)` and `read_status(query)`. It owns legal transitions, idempotency receipts, compare-and-set revisions, explicit external-operation intents, provider reconciliation, Human authority, Project isolation, and Controller-version fencing.

Why it fits the first slice: status is direct, the smallest useful state model is explicit, local atomicity is easy to state, and the difficult GitHub crash boundary is handled without turning projection rebuild and event-schema evolution into required first-slice contracts.

### 2. Append-only event-log engine — rejected as the selected primary model

An ordered per-run event stream is authoritative; a deterministic projector derives status and can rebuild it. This has strong replay and audit semantics and makes immutable history explicit. It also introduces projection freshness, event schema evolution, rebuild, and migration machinery at the foundation boundary.

Selected ideas retained: immutable transition/evidence envelopes with command identity and revisions; explicit distinction between external intent, verified success, and ambiguity; deterministic replay fixtures. Event sourcing is not required as the current-state authority for the next task.

## Matt scoring

| Criterion | State transition | Event log | Decision evidence |
| --- | ---: | ---: | --- |
| Domain clarity | 5 | 5 | Both assign distinct responsibilities to core terms. |
| Interface depth and locality | 5 | 5 | Both centralize workflow rules behind a small Module Interface. |
| Failure-case coverage | 5 | 5 | Both explicitly handle required crash, duplicate, authority, and version scenarios. |
| Simplicity | 4 | 2 | Snapshot authority avoids mandatory projection and event-schema lifecycle machinery. |
| Testability | 5 | 5 | Adapter-level fault injection and Interface-level scenarios are direct. |
| Agent executability | 4 | 4 | Explicit commands and gates bound actions; event streams add extra concepts. |
| Operational safety | 5 | 5 | Both block ambiguity and fence mutations. |

Scores use the rubric in `scoring-rubric.md`; no candidate was chosen by line count or visual polish. Safety and failure coverage were treated as gates.

## Important disagreements within the Matt run

1. **Authority:** current snapshot versus event stream. The selected model favors direct status and simpler precondition checks; the event candidate favors immutable replay authority.
2. **Audit:** transition records as audit/evidence versus all domain history as events. The selected model keeps the audit record but avoids requiring every status read to depend on projection health.
3. **Failure repair:** snapshot plus explicit recovery versus projection rebuild as a normal operation. The selected model permits replay fixtures and consistency checks without making rebuild the primary operational path.
4. **Complexity placement:** both use deep Interfaces, but the event candidate moves more lifecycle complexity into the first durable contract.

## Ideas selected for the next Kamy-Bot task

1. A deep Workflow engine Module, not a pass-through repository or framework wrapper.
2. Explicit command vocabulary; no generic state patches.
3. Durable idempotency receipts and expected-revision compare-and-set.
4. Current state, transition records, evidence, and external-operation intents committed atomically.
5. GitHub as an Adapter and shared external work record, never the Workflow database.
6. Marker-based GitHub reconciliation after intent commit; ambiguity becomes `Blocked`, never a guessed success or duplicate.
7. Project identity and Controller version in every durable mutation and status scope.
8. One active Controller-version lease per Project with fencing; no unscoped global scanner.
9. Ownership authority and later Scope approval remain explicit Human gates; Worker output cannot satisfy them.
10. Read-only status performs no write, retry, lease acquisition, or external call.
11. Deterministic replay fixtures over transition/evidence records, without selecting an external workflow framework.

## Rejected ideas

- Full event sourcing as the first-slice authority.
- GitHub labels/comments as the state machine.
- In-memory Controller state with periodic snapshots.
- Generic CRUD Adapter exposed to every caller.
- Generic event bus without ordered per-run semantics.
- Synchronous GitHub call inside a long local transaction.
- Framework-first selection based on familiarity.
- One global run scanner that ignores Project/version scope.

Reasons and edge behavior are recorded in both candidate artifacts and `candidate-evaluation.md`.

## Verification evidence

Evidence produced by this run is design evidence only:

- Both structurally different candidate artifacts exist under `docs/experiments/mattp/`.
- The selected proposal contains the state model, Module Interface, invariants, three Adapter seams, all required failure scenarios, verification matrix, scope boundary, rubric, rejected alternatives, and unresolved decisions.
- The evaluation records scores and selection rationale.
- The questions/assumptions artifact records that no blocking question was needed and separates context-grounded answers from unresolved decisions.
- Exact baseline commit is recorded in every artifact.

No behavioral test or smoke test was executed because this run is explicitly design-only and must stop before implementation. The selected proposal's verification matrix is the required pre-implementation contract, not evidence that production behavior already exists.

## Pstack comparison boundary

This Matt run did not read `docs/experiments/pstack/`, `docs/experiments/legacy/`, or any prior proposal. It therefore makes no claim about pstack's selected ideas, rejected ideas, disagreements, or skills. The cross-run comparison must be completed by the coordinator after the separate pstack run, using the same clarification record and scoring rubric. No pstack skill is recommended for porting based on this run alone.

## Unresolved decisions blocking production implementation

1. Durable-state technology and guarantees for atomicity, uniqueness, CAS, backup, retention, and restore.
2. Project ownership policy and authenticated Human decision model.
3. Safe deterministic GitHub marker and issue body/provenance contract.
4. Lease expiry, fencing, cutover, and recovery procedure for Controller versions.
5. Changed-source conflict handling: permanent block or authorized fork.
6. Read-only status freshness, redaction, authorization, and compatibility guarantees.
7. Retention, encryption, and redaction of request/provider/Human evidence.
8. Later risk classification and Delivery approval policy.
9. Explicit in-flight Controller-version migration policy, if ever needed.
10. Learning workflow inputs and approval contract for policy/capability-pack releases.

## Stop condition

The Matt side is complete at design artifacts. No production implementation begins from this run until the cross-run comparison is recorded and approved.
