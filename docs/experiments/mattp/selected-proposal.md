# Matt selected proposal: transactional Workflow engine

- Starting commit: `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`
- Selection source: Matt candidate comparison in `candidate-evaluation.md`.
- Status: design-only proposal. Stop before production implementation.

## Decision and scope

Select a deep **Workflow engine Module** backed by a durable current-state snapshot plus immutable transition and evidence records. The snapshot is authoritative for current state and command preconditions. Transition records are the audit trail and replay-fixture input; they are not a mandatory event-sourcing projection.

The first slice is exactly:

```text
Intake source
→ Work request
→ GitHub issue
→ ownership check
→ durable Workflow run
→ read-only status
```

The design supports one Project profile, one GitHub repository, and one Intake source. It leaves seams for Discovery, Triage, Scope approval, Worker execution, review, risk checks, delivery approval, and merge. It does not design those later phases in detail.

The two-gate autonomy model is explicit:

- **Gate 1 — authority/ownership:** the request is shown to belong to this Project under its ownership policy. An authorized Human decision is required wherever policy requires human authority. Passing this gate yields `ReadyForDiscovery`, not implementation approval.
- **Gate 2 — Approved scope:** later Discovery/Triage produce Approved scope and an authorized Human approves it before any Worker receives an implementation action. This gate is outside the first slice but is a required extension point.

High-risk Delivery approval remains a separate later gate. The Learning workflow remains separate from the main Workflow: it examines runs, proposes policy or capability-pack changes, tests them, and promotes or rolls back approved releases.

## Questions, answers, and assumptions

No blocking clarification question was necessary. The shared context and experiment brief answer the first-slice and authority questions. The following assumptions are conservative and must not be mistaken for approved product decisions.

| Question | Answer used |
| --- | --- |
| What is one duplicate? | Same `(Project, Intake source, stable source item ID)` is one Work request. Exact retries return the original result. A changed digest is an explicit Intake conflict, never a silent overwrite. |
| Does Intake decide scope? | No. Intake supplies request content and source identity only. |
| What does ownership acceptance mean? | Permission for this Project's Workflow to own the request. It does not mean Scope approval or Worker authorization. |
| Is one issue created per request? | Yes in this experiment. One Work request has one GitHub issue link. Child issues are later. |
| Is GitHub authoritative? | No. Durable Workflow state is authoritative; GitHub is an external shared record behind an Adapter. |
| Are local state and GitHub atomic? | No. Durable intent is committed before the provider call, and the result is committed after reconciliation. |
| What if GitHub outcome is ambiguous? | Keep the run `Blocked` with evidence. Reconcile by deterministic marker; do not create a possible duplicate. |
| Can two Controller versions process the same Project? | No. A durable active-version lease and fencing/version-scoped mutations permit one active version per Project. |
| What does status do? | It only reads durable records and returns an honest snapshot. It does not trigger work or external calls. |
| Which durable technology is selected? | None. Selection is deferred until atomicity, uniqueness, CAS, durability, backup, retention, and recovery requirements are approved. |

Additional assumptions:

1. Every run belongs to exactly one Project and one Controller version namespace.
2. Every command carries a caller-generated idempotency key and expected run revision.
3. A local transaction can atomically commit the state snapshot, one transition record, evidence links, external intent or result, and command receipt. It cannot include a GitHub call.
4. Transition records and evidence are append-only or explicitly superseded; no Human decision or provider result is silently rewritten.
5. A restart has no required in-memory continuation. It discovers pending work from durable state and intents.
6. The exact ownership policy, actor authentication, marker format, lease timing, recovery operator, and status freshness contract are unresolved.

## Domain terms and relationships

- **Project profile:** Project-specific repository, ownership policy, workflow labels, Intake sources, and integrations.
- **Project:** owns policy, Work requests, and the durable namespace. Project identity is mandatory; a GitHub issue ID alone is insufficient.
- **Intake source:** external source that supplies normalized request content and source identity. Its Adapter does not decide scope or Workflow state.
- **Work request:** user-originated unit of work. In the first slice, it links one stable source item, one Project, one Workflow run, and one GitHub issue.
- **GitHub issue:** shared external work record linked to the request. It is evidence/projection, not the Workflow state machine.
- **Workflow run:** one recorded attempt to move one Work request through ordered Workflow states.
- **Controller version:** one released Controller implementation. It owns a run namespace and is part of mutation authorization, not only diagnostic metadata.
- **Human decision:** authenticated, authorized decision for ownership or exceptional recovery. It cannot be inferred from Intake content or Worker output.
- **Evidence:** observable check, external response, durable outcome, or Human decision attached to a run. Evidence supports a decision; it does not pass a Human gate by itself.
- **External-operation intent:** durable instruction to reconcile a provider effect. Intent is not success.
- **Command receipt:** durable result for a command idempotency key. It returns the original result on retry.

Relationship:

```text
Project profile → Project
Project → Work request → Workflow run
Work request → one GitHub issue link
Workflow run → current state + revision
Workflow run → transition records + evidence + operation intents + command receipts
Controller version → owns Project run namespace and mutation lease
```

## Smallest useful state model

The authoritative `WorkflowRun.state` is:

| State | Meaning | Allowed next states |
| --- | --- | --- |
| `Received` | Work request and run are durably accepted; no issue link is committed | `IssuePending`, `Blocked` |
| `IssuePending` | An intent to ensure the one GitHub issue is committed | `IssueLinked`, `Blocked` |
| `IssueLinked` | One issue marker and reference are verified | `OwnershipPending`, `Blocked` |
| `OwnershipPending` | Ownership evidence or required Human authority is outstanding | `ReadyForDiscovery`, `Rejected`, `Blocked` |
| `ReadyForDiscovery` | Gate 1 passed; later Discovery may be scheduled | later states only |
| `Rejected` | Authorized ownership decision rejected this Project's responsibility | none in first slice |
| `Blocked` | Safe progress is impossible or an external outcome is ambiguous | explicit recovery to a prior actionable state only |

A transition record contains `from_state`, `to_state`, `run_revision`, command identity, idempotency key, actor kind/identity, reason, evidence references, and timestamp. A committed transition increments revision exactly once. `ReadyForDiscovery` must never be represented as Scope approval.

## Main Module and Interface

### Workflow engine Interface

Conceptual Interface:

```text
submit(command, idempotency_key) -> CommandResult
read_status(status_query) -> WorkflowStatus
```

First-slice commands are explicit discriminated commands, not generic field patches:

- `AcceptIntake(Project, source, source_item, content, digest, ControllerVersion)`
- `EnsureIssue(run, expected_revision)`
- `EvaluateOwnership(run, expected_revision)`
- `AcceptOwnership(run, expected_revision, HumanDecision)`
- `RejectOwnership(run, expected_revision, HumanDecision)`
- `RecoverBlockedRun(run, expected_revision, authorized_recovery_decision)`

The engine validates Project profile, Controller version, authorization, legal current state, expected revision, and idempotency; atomically commits local effects; calls a provider only for a committed external intent; then commits the provider result or durable failure. Callers never set state directly, write transition records, mutate projections, or bypass the engine with GitHub labels/comments.

### Interface invariants

1. **Atomic local transition:** state, revision, transition record, evidence links, intent/result, and receipt are all visible or none are visible.
2. **Compare-and-set ordering:** stale expected revisions fail without overwriting a newer transition and return current status.
3. **Legal transitions only:** the engine owns the state graph.
4. **Idempotent commands:** one key has one result within retention policy; payload reuse with different content is an error.
5. **External reconciliation:** a provider call is not success until one deterministic marker/reference is verified.
6. **Append-only evidence:** evidence is added or explicitly superseded; Human decisions are never silently replaced.
7. **Human authority:** only authorized actors satisfy ownership/rejection/recovery decisions.
8. **Project isolation:** all reads, writes, uniqueness, and status queries require Project identity.
9. **Controller isolation:** only the recorded version holding the active Project lease may mutate its run namespace.
10. **Read-only status:** `read_status` creates no receipt, lease, retry, transition, or provider call.
11. **No implicit rollback:** failures become `Blocked`; recovery needs an explicit authorized command.
12. **One issue per Work request:** durable uniqueness and marker reconciliation prevent duplicate issue links.

This is a deep Module. Its small Interface hides state legality, idempotency, recovery, external-operation reconciliation, authority, and version fencing. The deletion test predicts that removing it would distribute these rules across Controllers, Workers, recovery tools, and Adapters. The Interface is the test surface, producing leverage for callers and locality for maintainers.

## Adapter seams

Each seam has a provider Adapter and a deterministic fake, so each is a real variation point.

### Intake source Adapter

```text
read(source_item_ref) -> IntakeEnvelope
```

The normalized envelope contains source ID, stable source item ID, content, content digest, source revision, and observation time. It does not deduplicate, decide ownership, acknowledge the source, or advance state.

### GitHub Adapter

```text
ensure_issue(issue_spec, idempotency_key, marker) -> IssueResult
read_issue(issue_ref) -> ObservedIssue
```

`ensure_issue` finds the deterministic marker before creating. It verifies exactly one matching issue and returns typed outcomes for success, rate limit, authorization error, not found, and ambiguity. It does not mutate durable Workflow state. The marker contains safe Project/Work-request provenance without sensitive Intake content.

### Durable state Adapter

```text
transaction(work) -> committed_result
read_run(Project, ControllerVersion, run_id) -> DurableSnapshot
read_status_index(Project, ControllerVersion, filters) -> DurableSnapshots
```

The Adapter provides atomic transactions, uniqueness, CAS revision, receipts, intents, transition records, evidence, leases/fencing, and scoped reads. It need not expose CRUD methods to callers; Workflow semantics remain in the engine Implementation.

## First-slice protocol

1. Controller reads the configured Intake source.
2. `AcceptIntake` validates the Project and source identity. One transaction creates the Work request, `Received` run, initial transition, and receipt, or returns the exact duplicate's existing receipt/result.
3. `EnsureIssue` commits `Received → IssuePending` and an external intent before calling GitHub. The Adapter reconciles by marker/key. A second transaction commits the verified issue and `IssuePending → IssueLinked`.
4. `EvaluateOwnership` records automated evidence and commits `IssueLinked → OwnershipPending` when authority remains outstanding.
5. An authorized Human submits `AcceptOwnership` or `RejectOwnership`, yielding `ReadyForDiscovery` or `Rejected`.
6. `read_status` returns current state, revision, linked issue, evidence, pending intent, failure detail, and Controller version. It starts no later phase.

## Required failure behavior

### Duplicate intake

Concurrent exact deliveries race on a durable uniqueness constraint. Exactly one Work request/run/receipt exists. Losers return the winner's result. A changed digest for the same source identity becomes an explicit conflict and `Blocked` outcome; it cannot overwrite or create a second issue. Equal text from distinct sources is not cross-source deduplication.

### Controller restart

- Before local commit: no command effect is visible; retrying the same key is safe.
- After intent commit and before GitHub: run remains `IssuePending`; restart retries marker-safe reconciliation.
- After GitHub creation and before result commit: marker lookup returns the existing issue; no duplicate is created.
- After result commit: receipt/status returns the same result; no provider call is needed.
- During ownership: durable `OwnershipPending` remains visible; restart does not infer a Human decision.

### Partial write

A failed durable transaction exposes no partial state, transition, evidence, intent, or receipt. A later transition cannot appear without its predecessor. An ambiguous GitHub outcome is recorded as blocked evidence, not success. No failure silently rolls back history.

### Worker stop between transitions

Workers are not launched here. Later Workers must call `submit`. If a Worker commits T1 and stops before T2, status ends at T1 and no T2 is invented from a heartbeat or untrusted log. Retrying T1 with its receipt is idempotent; submitting T2 requires the current expected revision and legal state. External effects require Adapter reconciliation before retry.

### Controller version separation

A Project has one active-version lease with fencing. Every run, receipt, intent, transition, marker, and status index includes Project and Controller version. The old version becomes stopped/read-only before a new version acquires the lease.

- v1 mutates only v1 runs while fenced as active.
- v2 mutates only v2 runs after cutover.
- Cross-version mutation returns `VersionMismatch`.
- v2 does not scan all GitHub issues or claim v1 runs.
- In-flight migration is a later Human-approved capability, not part of this slice.

## Observable verification plan

This is planned evidence, not executed verification. Use deterministic fake Adapters and fault injection at every transaction/provider boundary. Observe `CommandResult`, `read_status`, durable records, and Adapter call counts.

| Scenario | Required observable evidence |
| --- | --- |
| Golden path | One request/run/issue; ordered states reach `ReadyForDiscovery`; status revision and evidence match durable records. |
| Exact duplicate | Concurrent retries return one run ID and receipt; one issue ensure path only. |
| Changed digest | Both digests and conflict reason visible; no silent overwrite or second issue. |
| Restart boundaries | Each pre/during/post call boundary resumes from committed state and returns the same result. |
| GitHub crash after create | Marker finds the original issue; exactly one link is committed. |
| Ambiguous GitHub | Run is `Blocked`; no guessed issue reference or duplicate create. |
| Partial local commit | State/transition/receipt are all-or-nothing. |
| Unauthorized ownership | No transition; authorized Human decision creates exactly one acceptance/rejection. |
| Worker stop | Status ends at last committed transition; retry cannot duplicate or skip. |
| Stale command | CAS conflict leaves newer state unchanged. |
| Two versions | Scoped populations remain separate; cross-version mutation is rejected; no simultaneous lease. |
| Read-only status | Repeated reads leave durable write count and all external Adapter call counts unchanged. |

## First slice and later capabilities

### Required now

One Project profile, one repository/source, normalized intake identity, one Work request per stable source item, one issue ensure/reconcile path, ownership evidence and Human gate, durable run snapshot and transition/evidence records, idempotency, CAS ordering, restart/partial-write handling, Controller-version isolation, and read-only status.

### Explicitly later

Discovery, Triage, child issues, detailed Scope approval and Approved scope materialization, Worker scheduling/capability packs/workspaces, pull requests, review, risk checks, Delivery approval, merge, provider acknowledgements, multiple source coordination, Controller migration, queue choice, multi-region behavior, and Learning workflow policy release mechanics.

## Scoring

| Criterion | Score | Rationale |
| --- | ---: | --- |
| Domain clarity | 5 | Terms and ownership relationships are explicit. |
| Interface depth and locality | 5 | A small engine Interface centralizes legality, idempotency, authority, recovery, and fencing. |
| Failure-case coverage | 5 | Required duplicate, restart, partial-write, Worker-stop, ambiguity, stale-command, and version cases have outcomes. |
| Simplicity | 4 | Snapshot reads are direct; receipts/intents are justified by external crash safety. |
| Testability | 5 | Interface-level deterministic Adapters and fault injection cover observable behavior. |
| Agent executability | 4 | Explicit commands and gates bound work; recovery/ownership configuration remains consequential. |
| Operational safety | 5 | No implicit authority, duplicate external mutation, unscoped Controller population, or silent failure. |

## Rejected alternatives

- **Event log as primary authority:** valuable replay/audit model, but projection freshness, event schema evolution, and rebuild/migration become mandatory first-slice machinery. Keep immutable transition/evidence records and replay fixtures without requiring event sourcing.
- **GitHub as Workflow state:** cannot provide local atomicity, receipts, Project/version fencing, or pre-call intent.
- **In-memory state with snapshots:** restart can lose command boundaries and duplicate provider effects.
- **Generic CRUD exposed to every caller:** shallow Module; invariants leak into all callers.
- **Synchronous GitHub inside a long transaction:** still lacks distributed atomicity and holds local resources across provider latency.
- **Global unscoped scanner:** permits two Controller versions or Projects to process overlapping populations.
- **Framework-first design:** framework selection is an Adapter/Implementation choice behind Kamy-Bot Interfaces.

## Unresolved decisions before implementation

1. Durable technology and its atomicity, uniqueness, backup, retention, and restore guarantees.
2. Exact ownership policy and Human authentication/authorization evidence.
3. Safe GitHub marker fields, issue body contract, and handling of issue deletion/repository transfer.
4. Lease expiry, fencing token, cutover, and dead-version recovery procedure.
5. Changed-source conflict policy: permanent block or explicit Human-authorized fork.
6. Status freshness, redaction, access control, and compatibility guarantees.
7. Retention, encryption, and redaction of request content, provider evidence, and Human decisions.
8. Later risk classification and Delivery approval policy.
9. Explicit Controller-version migration, if ever needed.
10. How Learning workflow proposals consume evidence without mutating production policy without approval.

No production code should be written until this proposal and the comparison result are approved.
