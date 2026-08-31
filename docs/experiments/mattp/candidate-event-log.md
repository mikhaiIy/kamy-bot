# Matt candidate: append-only Workflow event log

- Starting commit: `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`
- Run: Matt side of Experiment 001.
- Status: design-only candidate; no production code, GitHub mutation, or production Worker launch.

## Position and scope

This candidate makes an append-only Workflow event log the authoritative record. A deterministic projector folds the log into a read model for read-only status. The event log is not a generic audit trail added beside mutable state: accepted commands become ordered domain events, and current status is derived from those events. GitHub operations use durable intent and result events so a Controller restart can resume or reconcile a non-transactional external effect.

The fixed first slice is:

```text
Intake source → Work request → GitHub issue → ownership check → durable Workflow run → read-only status
```

It supports one Project profile, one GitHub repository, and one Intake source. It defines extension points for later Discovery, Triage, Scope approval, implementation, review, risk check, delivery approval, and merge without designing those phases in detail.

## Questions and conservative answers

| Question | Answer used by this candidate |
| --- | --- |
| What identifies a duplicate intake? | `(project_id, intake_source_id, source_item_id)` is a stable stream identity. A repeated delivery with the same content digest returns the existing command result. A changed digest is an explicit conflict event and never silently overwrites the request. |
| Is the event log event sourcing? | Yes, for the Workflow run stream: the ordered event stream is authoritative, and status is a projection. This is a deliberate candidate for comparison, not a framework selection. |
| Can an event mean that GitHub already changed? | Only a `GitHubIssueEnsured` event with a verified issue reference means success. `GitHubIssueIntentRecorded` is not success. Ambiguous outcomes remain pending or blocked until marker reconciliation proves one issue. |
| Who can pass ownership? | Automated checks can append evidence. An authorized Human must append the ownership acceptance or rejection event when the Project policy requires human authority. |
| What are the autonomy gates? | Gate 1 is ownership/authority before the run can enter `ReadyForDiscovery`. Gate 2 is Scope approval before a later Worker can execute Approved scope. High-risk Delivery approval is a later gate, not part of this slice. |
| What does read-only status do? | It reads the versioned projection only. It never appends events, acquires a lease, calls GitHub, retries intent, or advances a run. |
| Which event-store technology is selected? | None. The durable-state Adapter must provide ordered append, optimistic concurrency, idempotency, durable projections or rebuild, and scoped reads. |

## Assumptions

1. One Work request maps to one first-slice Workflow stream, one GitHub issue, and one Project namespace.
2. Event streams are scoped by Project and Controller version. A GitHub issue identifier alone never selects a stream.
3. Event append has an expected stream revision. A stale append fails without adding an event.
4. A command idempotency key is unique within Project, Controller version, command kind, and logical target. Reusing the key with a different payload is a deterministic conflict.
5. The event store can atomically append the event(s) for one local command and its command receipt. It cannot atomically append with GitHub.
6. The projector is deterministic and can rebuild status from the stream. Projection lag is reported in status rather than hidden.
7. A Controller version lease fences appends. A new version cannot append to an old version's stream without an explicit future migration protocol.
8. Event payloads are immutable. Corrections append compensating or superseding events; no prior Human decision or provider result is edited in place.
9. The exact ownership policy, actor authentication, event retention/redaction, GitHub marker format, and recovery operator remain unresolved.
10. Learning remains a separate workflow for policy and capability-pack proposals. It consumes run evidence but does not mutate this run stream by default.

## Domain terms and relationships

- **Project profile** describes one Project's repository, ownership policy, workflow labels, Intake sources, and integrations.
- **Project** owns the stream namespace, policy, Work requests, and Controller-version lease.
- **Intake source** supplies source identity and request content through an Adapter. It does not decide scope.
- **Work request** is represented by an immutable `WorkRequestAccepted` event and later issue-link and ownership events in the same stream.
- **GitHub issue** is an external shared work record. Its verified reference is captured as event data; GitHub does not own Workflow legality.
- **Workflow run** is the ordered stream of one attempt to move one Work request through the Workflow.
- **Controller version** scopes stream ownership, command receipts, event append authority, and status queries.
- **Workflow event** is an immutable fact accepted by the Workflow engine, with stream revision, event ID, actor, command identity, and payload.
- **Projection** is a deterministic read model derived from events. It is a query optimization and status representation, not the source of truth.
- **Command receipt** records the result of a command and the event IDs it appended. It makes retries return the same result.
- **External-operation intent** is a domain event saying that an external effect must be reconciled. It does not assert that the effect happened.
- **Evidence** is an immutable event payload or linked evidence record that supports a state decision. Evidence alone does not pass a Human gate.
- **Human decision** is an authenticated actor event that can satisfy an ownership gate where policy requires it.

Relationship:

```text
Project profile → Project
Project → (Work request stream, one Controller-version namespace)
Work request stream → ordered Workflow events
Workflow events → deterministic Projection → read-only status
Workflow stream → one GitHub issue reference after verified ensure
Controller version → owns/fences its stream and commands
```

## Smallest useful event/state model

The authoritative model is the event sequence. The projector exposes a compact derived state:

| Derived state | Required evidence | Permitted next event family |
| --- | --- | --- |
| `Received` | `WorkRequestAccepted` | `GitHubIssueIntentRecorded`, `IntakeConflictRecorded`, `RunBlocked` |
| `IssuePending` | `GitHubIssueIntentRecorded` without verified result | `GitHubIssueEnsured`, `ExternalOperationBlocked` |
| `IssueLinked` | `GitHubIssueEnsured` with one marker/reference | `OwnershipCheckRecorded`, `RunBlocked` |
| `OwnershipPending` | ownership check evidence; Human event may still be required | `OwnershipAccepted`, `OwnershipRejected`, `RunBlocked` |
| `ReadyForDiscovery` | authorized ownership acceptance | later-phase event families only |
| `Rejected` | authorized ownership rejection | none in this slice |
| `Blocked` | explicit failure/ambiguity reason | explicit `BlockedRunRecovered` followed by a legal event |

The projector computes state from the latest legal event sequence and stores `projected_revision`. It must reject a stream with an illegal sequence rather than manufacture a status. The event log records an event envelope containing event ID, stream ID, Project ID, Controller version, prior revision, new revision, event type, actor kind/identity, command ID, idempotency key, payload digest, and timestamp.

First-slice event families:

- `WorkRequestAccepted`
- `IntakeConflictRecorded`
- `GitHubIssueIntentRecorded`
- `GitHubIssueEnsured`
- `OwnershipCheckRecorded`
- `OwnershipAccepted`
- `OwnershipRejected`
- `RunBlocked`
- `BlockedRunRecovered`

These names describe domain facts, not provider callbacks. A provider response is evidence attached to an event. An event is accepted only after the engine checks that its predecessor state, authority, stream revision, and Project/version namespace are valid.

## Main deep Module and Interface

### Workflow event engine

Conceptual Interface:

```text
append(command, expected_revision, idempotency_key) -> CommandResult
read_status(project_id, controller_version, run_id) -> WorkflowStatus
rebuild_status(project_id, controller_version, run_id) -> WorkflowStatus
```

The first-slice command vocabulary is explicit:

- `AcceptIntake`
- `RecordIssueIntent`
- `RecordIssueResult`
- `RecordOwnershipCheck`
- `AcceptOwnership`
- `RejectOwnership`
- `RecoverBlockedRun`

The caller supplies the command, expected stream revision, and idempotency key. The engine loads the stream, folds it to current state, validates authority and legal next event, and atomically appends the event(s) plus a command receipt. `RecordIssueIntent` precedes the GitHub Adapter call. `RecordIssueResult` is accepted only with a verified marker/reference or an explicit blocked outcome.

The Interface is deliberately small. Event legality, stream folding, command deduplication, human authority, Controller fencing, projection invalidation, and external-operation recovery remain in the Implementation. Callers do not construct arbitrary events or patch projections. The deletion test passes: without this Module, each Controller, Worker, recovery tool, and projection would repeat stream validation and safety rules. The Interface is the test surface; tests submit commands, inject Adapter outcomes, and read status.

### Interface invariants

1. **Append-only authority:** accepted event payloads and envelopes are immutable; no update or delete can change Workflow history.
2. **Ordered optimistic concurrency:** an append names the expected stream revision; only one append can claim the next revision.
3. **Atomic command receipt:** the accepted event(s), new revision, and command receipt are committed together or none are visible.
4. **Idempotent retry:** a previously committed idempotency key returns its original result and event IDs; it appends nothing again.
5. **Legal event sequence:** commands can append only event families allowed by the folded state and Project policy.
6. **Human authority:** Human-only events require an authenticated authorized actor; Worker output and Intake content cannot synthesize one.
7. **Project and version isolation:** every stream, event, receipt, projection, and mutation is scoped to Project and Controller version; cross-version append is rejected.
8. **External truth is explicit:** intent, verified success, and ambiguity are different events. A call site is never treated as success.
9. **Deterministic projection:** equal event prefixes produce equal status. A projection revision behind the stream is reported as stale and never used to authorize a command.
10. **Read-only status:** status methods perform no append, retry, lease acquisition, or external call.
11. **No implicit rollback:** failures append `RunBlocked`; recovery requires an explicit authorized command and a legal subsequent event.
12. **One issue marker per request:** the stream may contain at most one verified issue reference. Reconciliation must find that marker before creating.

## Adapter seams

Two implementations at each seam—a provider implementation and deterministic fake—make these real seams.

### Intake source Adapter

```text
read(source_item_ref) -> IntakeEnvelope
```

`IntakeEnvelope` contains Project-resolved source ID, stable source item ID, content, content digest, source revision, and observation time. The Adapter does not deduplicate, decide ownership, or append events. No source acknowledgement is designed in this slice.

### GitHub Adapter

```text
find_issue(marker) -> IssueLookup
create_issue(issue_spec, idempotency_key, marker) -> IssueLookup
read_issue(issue_ref) -> ObservedIssue
```

The event engine commits `GitHubIssueIntentRecorded` before calling the Adapter. The Controller reconciles by deterministic marker. If exactly one issue is found, it submits `RecordIssueResult` with `GitHubIssueEnsured`. If creation succeeds but the process stops before result append, restart searches the marker. If the result is ambiguous, it appends `ExternalOperationBlocked`/`RunBlocked`; it never creates a second issue based only on timeout.

The GitHub Adapter translates rate limits, authorization errors, not-found responses, and ambiguous outcomes into typed observations. It does not append Workflow events or modify projections.

### Event-store and projection Adapter

```text
append(stream_id, expected_revision, event_batch, receipt) -> AppendResult
read_stream(stream_id) -> EventSequence
read_receipt(idempotency_scope, key) -> CommandResult?
read_projection(status_scope, run_id) -> WorkflowStatus
rebuild_projection(stream_id) -> WorkflowStatus
```

The Adapter provides durable atomic append and scoped reads. It may persist projections for fast status but must support rebuild or verification from the log. It enforces unique stream revision and receipt keys. It does not decide legal state transitions; the event engine does.

## End-to-end first-slice protocol

1. Controller reads an Intake source envelope.
2. `AcceptIntake` folds the stream identity lookup. An exact duplicate returns the existing receipt/result. A new request atomically appends `WorkRequestAccepted` and creates the stream at revision 1.
3. `RecordIssueIntent` appends `GitHubIssueIntentRecorded`; only after that event is committed does the Controller call GitHub with its marker and idempotency key.
4. Controller reconciles the Adapter observation. A unique verified issue appends `GitHubIssueEnsured`; an uncertain outcome appends `RunBlocked` with evidence.
5. `RecordOwnershipCheck` appends automated evidence and derives `OwnershipPending` when a Human decision is required.
6. An authorized Human appends `OwnershipAccepted` or `OwnershipRejected`, deriving `ReadyForDiscovery` or `Rejected`.
7. `read_status` reads the projection, verifies or reports its revision, and returns state, stream revision, issue reference, evidence, Controller version, and safe failure details. It starts no later phase.

A Controller restart resumes from the last event revision. It may retry only an uncompleted intent using the same marker and key. It never infers an event from process memory or an uncommitted provider call.

## Duplicate, restart, partial-write, Worker-stop, and version behavior

### Duplicate intake

Concurrent `AcceptIntake` commands use a unique stream identity and idempotency receipt. Exactly one `WorkRequestAccepted` event is appended. Repeated identical deliveries return its run ID and result. The same source item with a changed content digest appends `IntakeConflictRecorded` (or remains outside a run if the conflict occurs during initial reservation) and becomes visible as a blocked human decision; it never overwrites history or creates a second issue. Equal text from distinct sources is not deduplicated.

### Controller restart

- Before the intent append: no intent exists; retrying the same command key can append it.
- After intent append and before GitHub: the stream derives `IssuePending`; restart calls marker reconciliation.
- After GitHub mutation and before result append: marker search returns the existing issue; restart appends one verified result.
- After result append: retry returns the receipt and no provider call is required.
- During projection update: the stream is authoritative; status reports projection lag or rebuilds without changing the stream.

### Partial writes and external effects

A failed local append exposes no partial event batch or receipt. A failed projection update cannot invent a state; rebuild from the committed event prefix repairs it. A GitHub timeout is not a `GitHubIssueEnsured` event. A marker that resolves to zero or multiple issues leaves the run blocked with evidence. A later event append failure leaves the prior prefix authoritative. Event IDs and revisions make replay safe.

### Worker stopping between transitions

Workers are outside this slice, but later Worker commands must use `append`. If a Worker appends `T1` and stops before `T2`, the stream ends at `T1`; the projection reports exactly that prefix. Restart does not append `T2` from a heartbeat or external log. A retry with the same command key returns the T1 receipt; a new T2 command must meet the expected revision and legal-transition checks. If T1 represented an external effect, its Adapter must reconcile before another attempt.

### Separation between two Controller versions

The Project has one durable active-version lease with fencing token. A stream is named by `(project_id, controller_version, run_id)`, and every event append checks both version and fencing token. The old version must be stopped/read-only before the new version acquires the Project lease.

- `v1` appends only to v1 streams while holding its lease.
- `v2` appends only to v2 streams after cutover.
- A v2 command against a v1 stream returns `VersionMismatch`; status is also version-scoped by default.
- v2 does not scan all GitHub issues to claim v1 work.
- Migration of an in-flight v1 stream is a later explicit Human-approved protocol, not an accidental consequence of deployment.

Thus two Controller versions cannot process one issue population through a shared unscoped stream. Existing v1 history remains readable and auditable.

## Observable verification plan

This is a planned behavioral matrix, not executed test evidence. Use deterministic fake Adapters, fault injection at every append/provider boundary, and observe command results, event sequences, projected revisions, status, and external call counts.

| Scenario | Observable evidence |
| --- | --- |
| Golden path | One stream has ordered events for acceptance, intent, verified issue, ownership check, and authorized acceptance; status derives `ReadyForDiscovery`. |
| Exact duplicate intake | Concurrent deliveries yield one stream and one acceptance event; all retries return the same receipt/result. |
| Changed source digest | Conflict evidence is visible; no history is overwritten and no second issue exists. |
| Restart before/during/after GitHub | The event prefix identifies the safe resume point; marker reconciliation prevents duplicate issue creation. |
| Ambiguous GitHub result | No success event exists; status is `Blocked` with provider evidence and no guessed issue reference. |
| Partial local append | Event batch and receipt are all-or-nothing; projection can rebuild from the committed prefix. |
| Projection lag/rebuild | Status reports lag or rebuilds to the same revision without appending Workflow events. |
| Stale command | Expected-revision conflict appends no event and leaves the newer prefix unchanged. |
| Unauthorized ownership decision | Append is rejected with no Human event; authorized decision appends exactly one event. |
| Worker stop between transitions | Status ends at T1; retry cannot duplicate T1 or skip to T2. |
| Two Controller versions | Cross-version append is rejected; scoped status shows separate streams; lease history has no simultaneous owner. |
| Read-only status | Repeated reads leave event count, receipt count, projection revision, and Adapter call count unchanged. |

## First slice versus later capabilities

### Included in this candidate

- One Project profile, repository, and Intake source.
- Stable intake identity and Work request stream creation.
- Append-only durable Workflow events and deterministic projection.
- Exactly one GitHub issue ensure/reconcile path.
- Automated ownership evidence and explicit Human ownership gate.
- Command idempotency, expected stream revision, restart recovery, blocked ambiguity, and Controller-version fencing.
- Read-only status.

### Later and intentionally unspecified

- Discovery, Triage, child issues, and detailed Scope approval.
- Worker scheduling, capability packs, workspaces, pull requests, review, risk check, and merge.
- Delivery approval policy for high-risk changes.
- Learning workflow, Workflow auditor, Learning proposals, Policy releases, canary, and rollback details.
- Event schema evolution, projection-version migration, retention/redaction/encryption, multi-region operation, queue choice, and in-flight Controller-version migration.

The seam is extensible, but later capabilities must use the same engine Interface and cannot mutate projections or GitHub labels as a shortcut.

## Provisional scoring rubric

| Criterion | Score | Rationale |
| --- | ---: | --- |
| Domain clarity | 5 | Immutable events, derived state, Work request stream, evidence, intent, Human decisions, Project, and Controller version have explicit roles. |
| Interface depth and locality | 5 | `append`, status, and rebuild hide stream legality, receipts, projection, authority, and recovery. |
| Failure-case coverage | 5 | Restart boundaries, external ambiguity, partial append, projection lag, duplicates, Worker stop, and version fencing are explicit. |
| Simplicity | 2 | Event schema, projection rebuild/versioning, and stream operations add substantial first-slice machinery. |
| Testability | 5 | Event prefixes and deterministic projectors make replay, fault injection, and boundary assertions direct. |
| Agent executability | 4 | Ordered facts and command revisions bound actions, but projection/event-schema knowledge increases cognitive load. |
| Operational safety | 5 | Append-only history, explicit ambiguity, idempotency, Human gates, and version fencing prevent silent unsafe mutation. |

The candidate maximizes auditability and replay at the cost of a larger durable contract than the fixed slice strictly needs.

## Rejected alternatives

### Transactional current-state snapshot as primary authority

Rejected for this candidate because it makes history a secondary audit concern and makes projection/replay less direct. It remains the simpler competing candidate and should be scored separately rather than dismissed as unsafe.

### GitHub as Workflow state

Rejected because GitHub labels, comments, and issue search do not provide ordered per-run append, atomic receipts, Project/version fencing, or explicit external intent.

### In-memory event list with periodic export

Rejected because restart can lose the authoritative prefix or replay a provider mutation. The event log must be durable before the Controller relies on it.

### Generic event bus without a per-run stream

Rejected because delivery ordering, deduplication, expected revision, and read-only status would leak into every subscriber. The deep Module needs a durable per-run stream, not only transport.

### Synchronous GitHub call before recording intent

Rejected because a crash after GitHub creation and before local recording loses the reconciliation key. Intent must be durable first.

### Framework-first design

Rejected because the framework is an Adapter/Implementation choice behind Kamy-Bot's Workflow Interface; it cannot define Project policy or Human authority.

## Unresolved decisions before production implementation

1. Which event-store/durable-state technology supplies atomic append, unique stream revisions, backups, retention, and projection rebuild?
2. Which event types are public status contract versus private implementation evidence, and how are schemas versioned?
3. How are request content, Human decisions, and provider payloads redacted, encrypted, retained, and authorized?
4. What actor authentication and Project-specific policy determine whether ownership acceptance is Human-only or automation-eligible?
5. What exact GitHub marker is deterministic without exposing sensitive content, and how are repository transfer or issue deletion represented?
6. What lease expiry and fencing procedure safely cuts over Controller versions?
7. Does an Intake conflict remain blocked forever, or may an authorized Human append a fork/new Work request decision?
8. What projection freshness guarantee does read-only status provide, and what does a consumer do when status is stale?
9. How are event schema and projector migrations tested and rolled back?
10. What later risk policy requires the separate Delivery approval gate?

This document is a design comparison artifact only. No production implementation should begin from it without the comparison decision and approval.
