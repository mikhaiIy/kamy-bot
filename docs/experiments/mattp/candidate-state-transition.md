# Matt candidate: transactional state-transition workflow engine

## Position and scope

This candidate uses a durable current-state record plus explicit durable state-transition records. The current state is authoritative for execution and status; transition records are the audit trail and evidence for what the engine committed. This is not event sourcing: the engine does not reconstruct the current state by replaying an event log.

The candidate is deliberately limited to the Experiment 001 foundation slice:

```text
Intake source → Work request → GitHub issue → ownership check → durable Workflow run → read-only status
```

It models one Project profile, one GitHub repository, and one Intake source. The seams and state model leave room for Discovery, Triage, Scope approval, implementation, review, risk check, and merge, but do not design those phases in detail. No production code, GitHub mutation, or production Worker is implied by this document.

**Starting commit:** `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`

The design applies Matt vocabulary deliberately. The Workflow engine is a deep Module: callers submit a small command and receive a durable result, while transition legality, idempotency, recovery, external-operation reconciliation, and version isolation remain in the Implementation. Its Interface is also the test surface.

## Questions and conservative answers

| Question | Conservative answer or assumption |
| --- | --- |
| What is the unit of deduplication? | A stable `(project_id, intake_source_id, source_item_id)` identity. An exact repeated delivery returns the existing Work request and Workflow run. A changed content digest for the same stable item is an Intake conflict requiring an explicit human decision; it never silently overwrites or creates another issue. |
| Does the Intake source decide scope? | No. It supplies request content and source identity only. Scope authority remains a Human decision. |
| What does the first ownership check authorize? | It verifies that the request may enter this Project's workflow under its ownership policy. It does not approve implementation scope. Scope approval remains a later mandatory gate before implementation. |
| May an automated check approve ownership? | It may produce evidence and a recommendation. Where the policy requires human authority, only an authorized Human command can move the run to `ReadyForDiscovery`. |
| Is one GitHub issue created for one request? | Yes for this experiment. The issue link is durable and unique per Work request. Child issues are a later Discovery/Triage capability. |
| Is GitHub the workflow database? | No. GitHub is an external record and Adapter. Durable state is authoritative for the Workflow run; GitHub issue state is evidence and an external projection. |
| Can a database transaction include a GitHub call? | No assumption of distributed atomicity. The engine commits an external-operation intent, calls an idempotent Adapter operation, then commits the result in a separate short durable transaction. |
| What happens when an external provider cannot honor an idempotency key? | The Adapter must reconcile by a deterministic marker before retrying creation. If it cannot prove whether a mutation happened, the run remains `Blocked` for human recovery rather than creating a possible duplicate. |
| Can two Controller versions process one Project? | No. A durable active-version lease and version-scoped run queries allow one active version per Project. A version cannot mutate a run owned by another version. |
| What is the first read-only status? | A projection of durable records: Project, Work request, linked issue (if known), Controller version, current state, revision, pending intent, transition evidence, and safe failure information. It performs no writes. |
| Which database or queue is selected? | None. The durable-state Adapter is intentionally unspecified until the comparison and production design are approved. |
| What does restart guarantee? | No in-memory state is required. On restart, the Controller loads its version-scoped runs and reconciles pending intents using command receipts and provider idempotency. |
| What does a Worker do in this slice? | Nothing in production. The Worker-stop scenario is a contract test for the transition protocol that later Worker actions will use. |

## Domain terms and relationships

- A **Project profile** identifies one Project's repository, ownership policy, workflow labels, Intake sources, and external integrations. This candidate has exactly one configured GitHub repository and one configured Intake source.
- A **Project** owns the policy and all Work requests in this run namespace. Project identity is never inferred from a GitHub issue alone.
- An **Intake source** supplies an immutable source identity and request content. Its Adapter translates provider data into a normalized intake envelope.
- A **Work request** is the user-originated unit of work. It keeps the source identity, content digest, Project identity, and the durable link to exactly one first-slice Workflow run and (after provisioning) one GitHub issue.
- A **GitHub issue** is the shared external work record. It links product and workflow evidence, but it does not own transition legality.
- A **Workflow run** records one attempt to move one Work request through the Workflow. It belongs to one Project and one Controller version and has one current state and monotonic revision.
- A **Controller version** is the released Controller implementation that owns a run. It is part of the run's identity and mutation authorization, not merely diagnostic metadata.
- A **Human decision** is an explicitly recorded decision by an authorized actor. In this slice it can resolve ownership authority. It cannot be synthesized from an Intake message or Worker output.
- An **Evidence record** is an observable check, provider response, Human decision, or durable operation outcome attached to a run. Evidence supports status and audit; it does not by itself advance state.
- An **external-operation intent** is a durable instruction to reconcile a provider effect, such as ensuring the GitHub issue. It is not an event and is not itself a successful transition.
- A **Command receipt** records the result for a caller-provided idempotency key. It makes retries return the original result rather than reapplying a command.

The relationship is:

```text
Project profile
  └─ Project ── owns ── Work request ── has one ── Workflow run
       │                    │                         │
       │                    └─ links one GitHub issue │
       │                                              ├─ owned by one Controller version
       │                                              ├─ has current state + revision
       │                                              ├─ has transition records
       │                                              └─ has evidence and operation intents
       └─ configures one Intake source and one GitHub repository
```

A source item may be delivered more than once, but the stable source identity maps to one Work request. Distinct Intake sources do not become duplicates merely because their text is equal; there is no unsafe cross-source content deduplication in the first slice.

## Smallest useful state model

The authoritative `WorkflowRun.state` is one of the following:

| State | Meaning | Allowed next states |
| --- | --- | --- |
| `Received` | Intake was accepted and a Work request plus run were durably created; no issue link is committed yet. | `IssuePending`, `Blocked` |
| `IssuePending` | A durable intent says to ensure the one GitHub issue; the provider call may be pending, retrying, or awaiting reconciliation. | `IssueLinked`, `Blocked` |
| `IssueLinked` | The durable run has a verified issue reference and marker. | `OwnershipPending`, `Blocked` |
| `OwnershipPending` | Automated ownership evidence or required Human ownership authority is still outstanding. | `ReadyForDiscovery`, `Rejected`, `Blocked` |
| `ReadyForDiscovery` | Ownership authority has been accepted. This is the first-slice handoff point; Discovery is not implemented here. | later Workflow states only |
| `Rejected` | An authorized ownership decision rejected the request for this Project. Terminal in this slice. | none |
| `Blocked` | Progress is unsafe or cannot be reconciled automatically. It records a reason and recovery metadata. | the prior actionable state, only through an explicit recovery command |

`ReadyForDiscovery` does not mean Scope approval. It means only that ownership authority has passed. The full Workflow still requires Discovery, Triage, and Scope approval before any Worker receives Approved scope. Delivery approval remains a separate later Human gate only when risk policy requires it; merge is outside this slice.

A transition record stores `from_state`, `to_state`, `run_revision`, command identity, actor kind and identity, reason, evidence references, and timestamp. A committed transition increments the run revision exactly once. The state snapshot is authoritative, so status does not require replaying transition records.

## Main deep Module and Interface

### Module: Workflow engine

**Interface (conceptual, not source code):**

```text
submit(command, idempotency_key) -> CommandResult
read_status(status_query) -> WorkflowStatus
```

`command` is an explicit discriminated command, not a generic field patch. The first-slice commands are:

- `AcceptIntake(project_id, source_id, source_item_id, content, content_digest, controller_version)`
- `EnsureIssue(run_id, expected_revision)`
- `EvaluateOwnership(run_id, expected_revision)`
- `AcceptOwnership(run_id, expected_revision, human_decision)`
- `RejectOwnership(run_id, expected_revision, human_decision)`
- `RecoverBlockedRun(run_id, expected_revision, authorized_recovery_decision)`

`EvaluateOwnership` may only record automated evidence and move to a pending human gate when policy requires it. `AcceptOwnership`, `RejectOwnership`, and `RecoverBlockedRun` require an authorized Human decision. `read_status` is a query, not a command and must not create receipts, leases, retries, or provider calls.

Every command carries a caller-generated idempotency key. The key is scoped to Project, Controller version, command kind, and logical target. Repeating a key returns the original `CommandResult`, including the same run ID, state, revision, and issue reference. Reusing a key with a different payload is a deterministic idempotency error, never a second attempt.

The engine hides this sequence behind the small Interface:

1. validate Project profile, Controller version, command authorization, current state, and expected revision;
2. atomically reserve or read a Command receipt;
3. atomically apply a local state transition and its transition record, or atomically record an external-operation intent;
4. invoke the appropriate Adapter only for a committed intent;
5. atomically commit the provider result and next transition, or durable failure metadata;
6. return the committed result and make the same result visible through `read_status`.

The engine does not expose table layout, retries, queue mechanics, GitHub labels, or provider-specific errors to ordinary callers. Those belong in the Implementation and Adapters. That gives callers leverage: every caller receives the same ordering, idempotency, authorization, and recovery behavior. It gives maintainers locality: transition rules and failure policy have one home.

### Interface invariants

1. **Atomic local transition:** the current-state update, revision increment, transition record, evidence links, and command receipt are one durable transaction. Either all are visible or none are visible.
2. **Compare-and-set ordering:** a command names the expected run revision. A stale command cannot overwrite a newer transition; it receives a conflict result that includes current status.
3. **Legal transitions only:** the engine owns the state graph. Adapters and callers cannot set a state or patch a run directly.
4. **Idempotent commands:** a committed command key has one result forever within retention policy. Retry cannot duplicate a Work request, issue link, or transition.
5. **External effects are reconciled:** no provider operation is considered complete merely because the process reached the call site. Success requires a durable provider reference or evidence that a deterministic marker resolves to one reference.
6. **Monotonic evidence:** evidence is appended or superseded explicitly; the engine never silently rewrites a Human decision or provider result.
7. **Human authority:** only the configured owner or authorized actor can resolve ownership, rejection, or exceptional recovery. Worker output cannot satisfy this invariant.
8. **Project isolation:** every mutation and status query is scoped to a Project profile. A repository or issue ID without Project identity is invalid.
9. **Controller isolation:** a run can be mutated only by its recorded Controller version while that version owns the Project lease. A different version receives `VersionMismatch`.
10. **Read-only status:** `read_status` does not mutate durable state and does not call external Adapters. It reports unknown or stale external information honestly.
11. **No implicit rollback:** a failure does not move a run backward. `Blocked` plus an explicit recovery command is the only recovery route.
12. **One issue per Work request:** an issue marker and durable uniqueness constraint prevent two issue links for one request, including retries and restarts.

The deletion test supports this Module: deleting it would spread state graph validation, command deduplication, recovery, ownership checks, and version scoping across every Controller and Adapter caller. That complexity does not disappear, so the Module earns its seam. The Interface is the test surface: tests should submit commands and read status rather than inspect the database implementation.

## Adapter seams

Each Adapter sits at a real Seam where a Project-specific or provider-specific implementation can vary. The first implementation and a deterministic fake are two concrete Adapters, so these are real seams rather than speculative abstractions.

### Intake source Adapter

**Interface:** read a normalized envelope by source item reference.

```text
read(source_item_ref) -> { source_id, source_item_id, content, content_digest, source_revision, observed_at }
```

The Adapter does not decide scope, ownership, deduplication policy, or Workflow state. It may be an Obsidian daily-note Adapter later, but the engine sees only the normalized envelope. The source identity must be stable enough for deduplication. No source acknowledgement mutation is needed in this slice; acknowledgement would introduce another external recovery problem without advancing the required contract.

### GitHub Adapter

**Interface:** ensure and inspect the one issue represented by a deterministic request marker.

```text
ensure_issue(issue_spec, idempotency_key, marker) -> { issue_ref, observed_issue }
read_issue(issue_ref) -> observed_issue
```

`issue_spec` is derived from the Work request and Project profile. The marker contains Project identity, Work request identity, and Controller/run provenance that can be safely exposed in the issue metadata. `ensure_issue` first searches for the marker, then creates only if absent, and verifies that exactly one matching issue is returned. Provider retries use the same key and marker. Ownership policy evaluation consumes observed issue/request evidence but remains in the engine's Implementation, not in GitHub.

The Adapter translates rate limits, authorization failures, not-found responses, and ambiguous provider outcomes into typed outcomes. It must not advance the durable run itself.

### Durable state Adapter

**Interface:** one atomic transaction facility plus read operations:

```text
transaction(work) -> committed_result
read_run(project_id, controller_version, run_id) -> durable_snapshot
read_status_index(project_id, controller_version, filters) -> durable_snapshots
```

The transaction facility supports uniqueness constraints, compare-and-set on run revision, durable operation intents, command receipts, transition records, and evidence records. It may use a relational database or another implementation with equivalent atomicity; the candidate does not select one. The Adapter owns serialization, durability, and recovery of committed records, but the Workflow engine owns semantics.

The durable-state Seam is intentionally not a collection of public CRUD methods. A CRUD-shaped Interface would make every caller repeat invariants and would be a shallow Module. Transaction plus scoped reads hides those details and keeps the engine's state-transition behavior local.

## End-to-end first-slice protocol

1. The Controller reads the configured Intake source through its Adapter.
2. `AcceptIntake` validates the Project profile and source identity. One transaction either returns the existing receipt/run for an exact duplicate or creates the Work request, run in `Received`, initial transition record, and receipt.
3. The Controller submits `EnsureIssue`. The engine commits `Received → IssuePending` and an issue intent before calling GitHub. The GitHub Adapter reconciles by marker and idempotency key. A second transaction records the one issue reference and commits `IssuePending → IssueLinked`.
4. The Controller submits `EvaluateOwnership`. The engine reads provider evidence through the Adapter as needed, records the evidence, and commits `IssueLinked → OwnershipPending`. If policy permits a fully automated result, the result is still recorded explicitly; if policy requires owner authority, the run waits for a Human command.
5. An authorized Human submits `AcceptOwnership` or `RejectOwnership`. The engine records the decision and commits `OwnershipPending → ReadyForDiscovery` or `OwnershipPending → Rejected`.
6. `read_status` exposes the durable run and its evidence. It never starts Discovery, a Worker, a pull request, a review, risk classification, delivery approval, or merge.

The Controller may issue the next command only after the preceding `CommandResult` is committed. A restart can safely rediscover work from durable `IssuePending` or `OwnershipPending` records rather than relying on an in-memory continuation.

## Duplicate, restart, partial-write, and Worker-stop behavior

### Duplicate Intake

Two deliveries with the same Project, source, and stable source item ID race on a durable uniqueness constraint. Exactly one creates the Work request and run. Every loser reads and returns the winner's receipt/result; no second run or issue is created. The same source item with a different content digest is recorded as an Intake conflict and moves to `Blocked` (or remains blocked before run creation if the conflict is found during the reservation), with both digests visible to a Human. The engine never silently replaces the original request.

The same prose arriving from two distinct Intake sources is not automatically deduplicated. That conservative choice preserves source authority and avoids merging unrelated user intent.

### Controller restart

The Controller can stop after any command, before any Adapter call, during an Adapter call, or after an Adapter call and before the result transaction. On restart it uses `(project_id, controller_version)` to load only its own runs. It retries a committed `IssuePending` intent with the same marker/key, or presents `OwnershipPending` for the outstanding check/decision. It never infers a state from process memory or from an uncommitted call.

A committed Command receipt is returned directly on retry. A missing receipt means the local transaction did not commit; the command may be retried under the same key. A committed issue link means the engine does not create a replacement merely because GitHub is temporarily unavailable.

### Partial writes and external effects

- **Crash before a durable transaction commits:** no state, transition, receipt, or intent is visible. A retry starts from the prior committed state.
- **Crash after a local intent commits but before GitHub is called:** the run remains `IssuePending`; reconciliation safely calls `ensure_issue`.
- **GitHub creates an issue, then the process crashes before the result commit:** the next Adapter call finds the deterministic marker and returns the existing issue. The engine then commits the one link; no duplicate is created.
- **GitHub outcome is ambiguous:** the Adapter searches by marker. If it cannot establish a unique result, the engine records the ambiguity and moves to `Blocked`; it does not guess.
- **Crash after the result transaction commits:** retry returns the receipt and status shows `IssueLinked`; no provider call is needed.
- **A later state transition fails:** the prior state remains authoritative. There is no partially visible `IssueLinked → OwnershipPending` transition. Recovery starts from the last committed state.

### Worker stopping between two transitions

Workers are not launched in the first slice, but the protocol is designed for the later case. Suppose a Worker completes an action associated with transition `T1`, then stops before submitting `T2`. Only `T1` can be visible if its transaction committed. `T2` is absent from the current state and transition records. A Controller restart does not invent `T2` from a process heartbeat or an untrusted log; it presents the run at the last committed state and uses a command receipt/evidence check to decide whether a safe retry is possible. If an external effect may have happened, the relevant Adapter must reconcile by idempotency key before retry.

A Worker cannot atomically mutate the next state by changing a GitHub label or writing a local file. State advances only through the engine Interface, which preserves the same invariants for Worker commands as for Controller commands.

### Separation between two Controller versions

Every run, command receipt, operation intent, transition record, issue marker, and status index includes Project identity and Controller version. The Project profile has one durable active-version lease. Cutover is a control-plane Human decision: the old version is stopped or made read-only before the new version acquires the lease. This candidate does not define live migration.

- Controller `v1` can mutate only `v1` runs while holding the `v1` lease.
- Controller `v2` can mutate only `v2` runs after it acquires the Project lease.
- A `v2` command targeting a `v1` run is rejected with `VersionMismatch`; the inverse is also true.
- A new version does not scan all GitHub issues or all open Work requests. It receives only new intake after cutover, in its own run namespace.
- Status may be granted to an authorized observer, but status queries are version-scoped by default and never grant mutation authority.

Thus two binaries cannot process the same issue population by accidentally sharing an unscoped queue. Existing `v1` runs remain durable and auditable; an explicit, Human-approved migration protocol is a later capability.

## Observable verification plan

This is a planned behavioral verification matrix, not a report of executed tests. Each case drives the engine Interface with real or deterministic fake Adapters and observes `CommandResult`, `read_status`, durable records, and Adapter call counts.

| Scenario | Observable acceptance evidence |
| --- | --- |
| One Project, one source, golden path | One Work request, one run, one issue marker/reference, ordered transitions `Received → IssuePending → IssueLinked → OwnershipPending → ReadyForDiscovery`, and a status read containing the same revision/evidence. |
| Ownership rejection | Unauthorized actor is rejected without a transition; authorized rejection creates exactly one Human decision and `Rejected`; no later command can start implementation. |
| Duplicate exact intake | Concurrent/repeated submissions return one run ID and one receipt; durable uniqueness and GitHub Adapter call count show no second issue attempt. |
| Changed content under same source item | Both content digests are observable; the run is blocked for explicit review; no silent overwrite or second issue is visible. |
| Controller restart at every command boundary | Restarted Controller reconstructs pending work from durable records, returns an existing receipt where committed, and reaches the same final result without duplicate issue creation. |
| Crash before/after each durable commit | Fault-injected durable Adapter shows all-or-nothing local records; an intent without a result remains pending and is recoverable. |
| GitHub success followed by lost process | Marker reconciliation returns the original issue and commits one link; Adapter call history proves no duplicate issue. |
| Ambiguous GitHub result | Status is `Blocked` with an explicit ambiguity reason; no guessed issue link exists. |
| Worker stops between transitions | The status and transition sequence stop exactly at the last committed transition; a retry cannot skip a required transition or apply it twice. |
| Two Controller versions | `v1` and `v2` runs remain in separate status scopes; cross-version mutation returns `VersionMismatch`; lease history shows no simultaneous active owner. |
| Read-only status | Repeated `read_status` calls return durable snapshots while durable write count and all external Adapter call counts remain unchanged. |
| Stale command | A command with an old expected revision returns a conflict and leaves the newer state unchanged. |
| Human gate safety | Worker or Controller automation cannot submit a Human-only decision without actor evidence; scope approval is not present in first-slice status as if it had occurred. |

The useful proof is observable behavior, not source shape or line count. The same scenarios can be replayed against an in-memory durable Adapter and a production candidate once one is selected.

## First slice versus later capabilities

### Included now

- One Project profile with one repository and one Intake source.
- Intake normalization and stable source identity.
- Work request creation and exact duplicate handling.
- Durable Workflow run current state and transition records.
- One GitHub issue ensure/reconcile operation.
- Automated ownership evidence plus explicit Human ownership decision where required.
- Command idempotency, compare-and-set revision checks, restart recovery, partial-write handling, and Controller-version isolation.
- Read-only status projection.

### Explicitly later

- Multiple Intake sources and provider-specific source acknowledgement.
- Discovery and Triage, including child issues.
- Scope approval details and Approved scope materialization.
- Worker scheduling, capability packs, workspace execution, pull requests, review, risk check, and merge.
- Delivery approval for high-risk changes and merge policy.
- Learning workflow, Workflow auditor, Learning proposals, Policy releases, canaries, and rollback.
- Controller-version migration of in-flight runs, multi-region operation, queue selection, retention policy, and production database selection.

The first slice exposes seams that can carry these capabilities without pretending to solve them. In particular, later Worker commands must reuse `submit` and its transition invariants; they must not bypass the engine with direct provider writes.

## Provisional scoring rubric

Scores are provisional 1–5 and evaluate this candidate alone for the Experiment 001 question. They are not a comparison result against the sibling candidate.

| Criterion | Score | Rationale |
| --- | ---: | --- |
| Domain clarity | 5 | Work request, run, current state, transition record, evidence, intent, receipt, Project, and Controller version have distinct responsibilities and explicit relationships. |
| Interface depth and locality | 5 | Two engine operations hide state legality, idempotency, recovery, authority, and version isolation. The deletion test predicts substantial complexity would otherwise leak into callers. |
| Failure-case coverage | 5 | Duplicate delivery, restart timing, non-atomic external effects, ambiguous provider outcomes, stale commands, Worker stop, and version cutover are first-class behavior. |
| Simplicity | 4 | A current snapshot is simpler to query and reason about than replay-based state, but durable receipts and operation intents add necessary machinery around GitHub's non-transactional nature. |
| Testability | 5 | The engine Interface is the test surface; Intake, GitHub, and durable-state Adapters can deterministically inject retries, crashes, stale revisions, and ambiguous outcomes. |
| Agent executability | 4 | Explicit commands and state transitions bound Worker behavior well. Human authority and recovery policy still require carefully supplied Project configuration. |
| Operational safety | 5 | Human gates are explicit, external ambiguity blocks rather than duplicates, and controller leases/version-scoped records prevent two versions from sharing an issue population. |

The main trade-off is deliberate: more durable protocol state than a CRUD workflow buys predictable recovery and safety at the cost of storage and Adapter implementation work. That trade-off is appropriate for a load-bearing factory foundation.

## Rejected alternatives

### Pure event sourcing

Rejected as the primary model for this candidate. A full event log would make replay, projection versioning, event schema evolution, and duplicate command handling central before the first slice needs them. Durable transition records are retained for audit and evidence, but current state is authoritative so status and recovery remain local and inexpensive.

### GitHub as the state machine

Rejected because GitHub issue labels/comments are not a transactional Workflow store, cannot enforce Project and Controller-version uniqueness reliably, and cannot represent an uncommitted external-operation intent. It remains an Adapter and shared work record.

### In-memory Controller state with periodic snapshots

Rejected because a restart can lose the exact command boundary and duplicate a GitHub mutation. Durable state must precede recovery; memory may cache but cannot be authoritative.

### Generic CRUD repository exposed to every caller

Rejected as a shallow Module. It would force Controllers, Workers, and recovery tools to duplicate transition legality, CAS, idempotency, human authority, and version checks. The engine's transaction Interface provides more leverage and locality.

### Distributed queue as the primary contract

Rejected for the first slice. A queue can wake a Controller but does not define durable Workflow semantics, idempotency, external reconciliation, or read-only status. Queue choice can be an internal Implementation detail later.

### Synchronous GitHub call inside one long database transaction

Rejected because a provider timeout would hold a durable transaction open and still would not provide atomicity. The committed intent plus idempotent Adapter reconciliation handles the unavoidable two-system seam explicitly.

### One global active-run scanner

Rejected because an unscoped scanner makes Controller-version separation and Project isolation accidental. All scans and leases are scoped to Project and Controller version.

### Framework-first design

Rejected because choosing an external agentic framework would reverse the dependency: Kamy-Bot's Workflow and safety contracts must remain behind its own Interfaces and Adapters. Framework selection is a later Implementation decision.

## Unresolved decisions before production implementation

1. Which durable-state technology provides the required atomic transactions, uniqueness, compare-and-set, retention, and backup guarantees?
2. What exact fields and visibility rules form the deterministic GitHub issue marker without leaking sensitive Intake content?
3. Which actors and groups are authorized by each Project's ownership policy, and how are Human decisions authenticated and retained?
4. Does a changed source revision become a blocked conflict forever, or may an authorized Human explicitly fork a new Work request?
5. What lease expiry, fencing token, and operator procedure are required for safe Controller cutover and recovery from a dead version?
6. How are provider rate limits, issue deletion, repository transfer, and GitHub authorization changes represented without losing the durable run?
7. What retention, redaction, encryption, and access policy applies to request content, transition evidence, and Human decisions?
8. What later policy determines risk level and when the separate Delivery approval gate is required?
9. What explicit migration contract, if any, permits a new Controller version to take ownership of an in-flight old-version run?
10. Which portions of the transition record become externally queryable, and what compatibility guarantees will read-only status provide?

These decisions should be resolved and approved before production code begins. Until then, this candidate is a design comparison artifact only.
