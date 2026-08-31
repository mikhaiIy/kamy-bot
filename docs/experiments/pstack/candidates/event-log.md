# Candidate: append-only workflow event log with status projection

**Baseline commit:** `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`

This is a pstack candidate for Experiment 001. It designs only the first slice:

```text
Intake source → Work request → GitHub issue → ownership check → durable Workflow run → read-only status
```

## Usage (caller's view)

The public surface is a command gateway plus a read-only status query. Callers submit domain commands; they never append wire records, call GitHub directly, or mutate a status view. Every accepted command produces an append-only event batch, and the projection is derived from that log.

```ts
const receipt = await workflow.dispatch({
  kind: "AcceptIntake",
  commandId: "obsidian:daily:2026-08-31#request-17",
  projectId: "kamy-bot",
  controllerVersion: "controller-2026-08-31.1",
  source: { kind: "obsidian", identity: "daily:2026-08-31#request-17" },
  request: {
    title: "Add a status endpoint",
    body: "Expose the current workflow status without mutating the run.",
    fingerprint: "sha256:...",
  },
  observedAt: "2026-08-31T09:00:00Z",
});

const view = await workflow.readStatus(receipt.workflow);
// { phase: "intake", issue: "not_requested", ownership: "not_checked", ... }
```

The intake adapter normalizes its provider payload into `AcceptIntake`; the command gateway owns deduplication and appends `IntakeAccepted` (or `DuplicateIntakeObserved`). It does not expose Obsidian's note object to the domain.

### Call site: intake polling adapter

```ts
for await (const submission of intakeSource.poll(cursor)) {
  const receipt = await workflow.dispatch({
    kind: "AcceptIntake",
    commandId: submission.deliveryId,
    projectId: profile.projectId,
    controllerVersion: controller.version,
    source: submission.source,
    request: submission.request,
    observedAt: submission.observedAt,
  });
  cursor = submission.nextCursor;
  await intakeSource.acknowledge(submission.deliveryId, receipt.outcome);
}
```

A repeated delivery uses the same `commandId` and returns the original receipt. A different delivery with the same request fingerprint is recorded as a duplicate and points to the original workflow, rather than creating a second issue.

### Call site: controller reconciliation loop

```ts
const status = await workflow.readStatus({
  projectId: profile.projectId,
  workflowId,
  controllerVersion: controller.version,
});

if (status.issue === "not_requested") {
  await workflow.dispatch({
    kind: "EnsureGitHubIssue",
    commandId: `issue:${workflowId}:ensure`,
    workflow: { projectId: profile.projectId, workflowId },
    controllerVersion: controller.version,
  });
}

if (status.issue === "linked" && status.ownership === "not_checked") {
  await workflow.dispatch({
    kind: "CheckOwnership",
    commandId: `ownership:${workflowId}:check`,
    workflow: { projectId: profile.projectId, workflowId },
    controllerVersion: controller.version,
  });
}
```

`EnsureGitHubIssue` and `CheckOwnership` are commands. Their handlers append an intent event before an external call, then append the observed result. Re-running the loop is safe and does not require the loop to understand event-storage details.

### Call site: read-only status endpoint

```ts
export async function getStatus(request: Request): Promise<Response> {
  const ref = parseStatusReference(request); // domain reference, not a storage key
  const view = await workflow.readStatus(ref);
  return Response.json({
    workflow: view.workflow,
    phase: view.phase,
    issue: view.issue,
    ownership: view.ownership,
    run: view.run,
    gates: view.gates,
    observedThrough: view.observedThrough,
  });
}
```

The endpoint cannot append events. `observedThrough` makes projection freshness visible instead of pretending that an eventually consistent read is current.

## Shape

### Domain types

The event log is keyed by a workflow stream, not by a mutable aggregate snapshot. A stream is isolated by Project, Work request, and Controller version:

```ts
type ProjectId = string & { readonly __brand: "ProjectId" };
type WorkflowId = string & { readonly __brand: "WorkflowId" };
type ControllerVersion = string & { readonly __brand: "ControllerVersion" };
type CommandId = string & { readonly __brand: "CommandId" };
type EventId = string & { readonly __brand: "EventId" };
type Sequence = number & { readonly __brand: "Sequence" };
type Fingerprint = `sha256:${string}`;
type IsoTimestamp = string & { readonly __brand: "IsoTimestamp" };

type WorkflowRef = Readonly<{
  projectId: ProjectId;
  workflowId: WorkflowId;
  controllerVersion: ControllerVersion;
}>;

type IntakeSource = Readonly<{
  kind: string;
  identity: string; // stable provider identity, not a transport payload
}>;

type WorkRequest = Readonly<{
  title: string;
  body: string;
  fingerprint: Fingerprint;
}>;

type ProjectProfile = Readonly<{
  projectId: ProjectId;
  repository: Readonly<{ owner: string; name: string; defaultBranch: string }>;
  ownershipPolicy: string;
  workflowLabels: ReadonlyArray<string>;
  intakeSourceKind: string;
  policyRelease: string;
  autonomy: Readonly<{
    scopeApproval: "required-before-worker";
    deliveryApproval: "required-for-high-risk";
  }>;
}>;
```

`ProjectProfile` is loaded and pinned into the first event as a profile digest and policy-release identifier. A later profile edit cannot change the meaning of an already accepted workflow. The profile contains one repository and one intake source in this slice; multiple Projects remain independent streams.

### Commands

Commands are domain intent, not provider requests. The gateway validates a command at the boundary, assigns no hidden defaults, and routes it to the handler that owns the decision.

```ts
type WorkflowCommand =
  | Readonly<{
      kind: "AcceptIntake";
      commandId: CommandId;
      projectId: ProjectId;
      controllerVersion: ControllerVersion;
      source: IntakeSource;
      request: WorkRequest;
      observedAt: IsoTimestamp;
    }>
  | Readonly<{
      kind: "EnsureGitHubIssue";
      commandId: CommandId;
      workflow: Pick<WorkflowRef, "projectId" | "workflowId">;
      controllerVersion: ControllerVersion;
    }>
  | Readonly<{
      kind: "CheckOwnership";
      commandId: CommandId;
      workflow: Pick<WorkflowRef, "projectId" | "workflowId">;
      controllerVersion: ControllerVersion;
    }>
  | Readonly<{
      kind: "StartDurableRun";
      commandId: CommandId;
      workflow: Pick<WorkflowRef, "projectId" | "workflowId">;
      controllerVersion: ControllerVersion;
    }>;
```

`StartDurableRun` records a durable Workflow run and does **not** start a Worker. Worker execution, Discovery, Triage, Scope approval, implementation, review, merge, and delivery actions are later capabilities. The two-gate policy is nevertheless explicit now: Scope approval is required before any Worker starts; Delivery approval is required only for high-risk changes. The first slice records these policy facts and reports them as not-yet-reached gates.

### Events

Events are immutable domain facts. Provider response bodies, database rows, and framework objects are private to their adapters and are reduced to these facts before append.

```ts
type WorkflowEvent =
  | Readonly<{
      type: "IntakeAccepted";
      workflow: WorkflowRef;
      source: IntakeSource;
      request: WorkRequest;
      profileDigest: string;
      policyRelease: string;
      occurredAt: IsoTimestamp;
    }>
  | Readonly<{
      type: "DuplicateIntakeObserved";
      workflow: WorkflowRef;
      duplicateCommandId: CommandId;
      duplicateFingerprint: Fingerprint;
      occurredAt: IsoTimestamp;
    }>
  | Readonly<{
      type: "IssueLinkRequested";
      workflow: WorkflowRef;
      requestKey: string;
      occurredAt: IsoTimestamp;
    }>
  | Readonly<{
      type: "IssueLinked";
      workflow: WorkflowRef;
      issue: Readonly<{ number: number; url: string; repository: string }>;
      requestKey: string;
      occurredAt: IsoTimestamp;
    }>
  | Readonly<{
      type: "IssueLinkFailed";
      workflow: WorkflowRef;
      requestKey: string;
      retryable: boolean;
      reasonCode: string;
      occurredAt: IsoTimestamp;
    }>
  | Readonly<{
      type: "OwnershipCheckRequested";
      workflow: WorkflowRef;
      checkKey: string;
      occurredAt: IsoTimestamp;
    }>
  | Readonly<{
      type: "OwnershipVerified";
      workflow: WorkflowRef;
      policyRelease: string;
      evidenceRef: string;
      occurredAt: IsoTimestamp;
    }>
  | Readonly<{
      type: "OwnershipRejected";
      workflow: WorkflowRef;
      reasonCode: string;
      occurredAt: IsoTimestamp;
    }>
  | Readonly<{
      type: "WorkflowRunStarted";
      workflow: WorkflowRef;
      profileDigest: string;
      policyRelease: string;
      gates: Readonly<{
        scope: "required-before-worker";
        delivery: "required-for-high-risk";
      }>;
      occurredAt: IsoTimestamp;
    }>;

type EventEnvelope = Readonly<{
  eventId: EventId;
  stream: WorkflowRef;
  sequence: Sequence;
  commandId: CommandId;
  event: WorkflowEvent;
}>;
```

Intent events (`IssueLinkRequested`, `OwnershipCheckRequested`) make an external action durable before it runs. Result events make reconciliation observable. A command may append more than one event, but its batch is atomic. No event is edited or deleted; correction is a new fact.

### Durable append boundary

```ts
interface AppendOnlyEventLog {
  append(
    stream: WorkflowRef,
    expectedSequence: Sequence,
    commandId: CommandId,
    events: ReadonlyArray<WorkflowEvent>,
  ): Promise<AppendReceipt>;

  read(stream: WorkflowRef, after?: Sequence): Promise<ReadonlyArray<EventEnvelope>>;
  findByCommand(commandId: CommandId): Promise<AppendReceipt | null>;
}

type AppendReceipt = Readonly<{
  workflow: WorkflowRef;
  firstSequence: Sequence;
  lastSequence: Sequence;
  eventIds: ReadonlyArray<EventId>;
  replayed: boolean;
}>;
```

`append` is the only durable write. The storage transaction atomically inserts the event batch and the command-id result. `expectedSequence` rejects stale concurrent writers; `commandId` makes retries return the original receipt. A unique `(projectId, workflowId, controllerVersion, sequence)` key and a unique command-id key enforce this at storage level, not only in process memory.

The public gateway is intentionally small but deep:

```ts
interface WorkflowLog {
  dispatch(command: WorkflowCommand): Promise<DispatchReceipt>;
  readStatus(ref: WorkflowRef): Promise<StatusView>;
}

type DispatchReceipt = Readonly<{
  workflow: WorkflowRef;
  outcome: "accepted" | "duplicate" | "already_complete" | "blocked";
  append: AppendReceipt;
}>;
```

`dispatch` hides validation, stream sequencing, duplicate detection, controller fencing, adapter invocation, intent/result reconciliation, and projection invalidation. Callers do not coordinate `load → validate → save` stages. The interface is smaller than the complexity it hides, so it is deep rather than a pass-through facade.

### External adapter seams

Adapters receive domain inputs and return domain observations. They cannot append directly or expose provider wire types.

```ts
interface IntakeSourceAdapter {
  poll(cursor: string | null): AsyncIterable<Readonly<{
    deliveryId: CommandId;
    source: IntakeSource;
    request: WorkRequest;
    observedAt: IsoTimestamp;
    nextCursor: string;
  }>>;
  acknowledge(deliveryId: CommandId, outcome: "accepted" | "duplicate" | "blocked"): Promise<void>;
}

interface GitHubIssueAdapter {
  ensureIssue(input: Readonly<{
    repository: ProjectProfile["repository"];
    request: WorkRequest;
    requestKey: string;
    idempotencyKey: string;
  }>): Promise<Readonly<{
    issue: Readonly<{ number: number; url: string; repository: string }>;
    created: boolean;
  }>>;
}

interface OwnershipAdapter {
  check(input: Readonly<{
    repository: ProjectProfile["repository"];
    issue: Readonly<{ number: number; url: string; repository: string }>;
    policy: string;
    checkKey: string;
  }>): Promise<Readonly<{
    allowed: boolean;
    reasonCode?: string;
    evidenceRef: string;
  }>>;
}
```

`AcceptIntake` is the intake adapter's command boundary. `EnsureGitHubIssue` and `CheckOwnership` are the GitHub and ownership command boundaries. Their handlers append intent events, call the corresponding adapter with stable idempotency keys, and append exactly one normalized result event. Adapter retries are therefore commands producing facts, rather than hidden mutations in a projection.

### Module map

```text
workflow-log/                  public command and read surface
  gateway                      boundary validation, command idempotency, controller fence
  commands/intake              creates the initial workflow stream and duplicate fact
  commands/github-issue         owns issue intent/result reconciliation
  commands/ownership             owns policy check intent/result reconciliation
  commands/run                  records durable run; enforces pre-worker gate
  event-log                     atomic append, sequence checks, command receipts
  projection                    pure reducer plus materialized StatusView
  profile                       profile/policy pinning and repository ownership
  adapters/intake               provider polling and normalized submissions
  adapters/github               issue ensure and provider idempotency mapping
  adapters/ownership            ownership evidence and policy-provider mapping
  learning                      separate read/audit stream and proposal boundary
```

The modules are grouped by knowledge: event durability knows atomic append; command modules know the decisions and adapter contracts; projection knows how facts become status; profile knows policy pinning. They are not split into temporal `load`, `validate`, `transform`, and `save` stages. `gateway` is a genuine policy boundary, not a chain of pass-through methods.

### Domain relationships

```text
Project profile
  ├─ identifies one repository, ownership policy, labels, intake source
  ├─ pins controller version and policy release at acceptance
  └─ declares Scope and Delivery autonomy gates
       ↓
Intake source ──normalizes──> Work request ──accepted──> workflow event stream
                                                     │
                                                     ├─ issue intent/result ──> GitHub issue
                                                     ├─ ownership intent/result ──> ownership decision + evidence
                                                     └─ durable run ──> read-only status projection

Workflow event log ──reduce──> StatusView
Workflow event log ──read only──> Learning workflow ──propose/approve──> new policy release
```

One Work request has one workflow stream and one GitHub issue in this slice. A Workflow run is the durable record of this attempt; later retries do not create another run. A controller version is part of the stream identity and never merely a mutable field in a shared status row.

### State model

The reducer derives state from event order; it never accepts a caller-supplied status transition.

```text
(no stream)
   │ AcceptIntake
   ▼
intake accepted ──EnsureGitHubIssue──> issue requested ──IssueLinked──> issue linked
                                              │                         │
                                              └─IssueLinkFailed───────────┘
                                                                        │ CheckOwnership
                                                                        ▼
                                                               ownership requested
                                                               │              │
                                                   OwnershipVerified   OwnershipRejected
                                                               │
                                                   StartDurableRun
                                                               ▼
                                                          run durable
```

`DuplicateIntakeObserved` is an additional fact on the original stream and leaves the state unchanged. A failed issue link remains retryable only when its result says `retryable`; a non-retryable failure is visible as blocked. Ownership rejection is terminal for this first-slice run and requires a human decision outside this candidate to proceed.

```ts
type StatusView = Readonly<{
  workflow: WorkflowRef;
  phase: "intake" | "issue" | "ownership" | "run" | "blocked";
  issue: "not_requested" | "requested" | "linked" | "failed";
  ownership: "not_checked" | "requested" | "verified" | "rejected";
  run: "not_started" | "durable";
  gates: Readonly<{
    scope: "required-before-worker" | "not_reached";
    delivery: "required-for-high-risk" | "not_reached";
  }>;
  observedThrough: Sequence;
}>;
```

The `run: "durable"` state means that the event exists durably. It must not be read as permission to start a Worker. Later phases add `ScopeApproved`, Worker, review, risk, and delivery events to the same event vocabulary only after this slice is approved.

### Controller version fencing

The controller claims a Project/version lease in a separate control stream before dispatching commands. The gateway verifies that the command's `controllerVersion` is the active version. Every workflow stream, event envelope, projection key, and idempotency key includes that version. A stale controller receives `ControllerVersionNotActive` and cannot append, even if it has a cached status view.

A deployment handoff is therefore:

1. Stop dispatch from the old version.
2. Persist the active-version change and lease epoch.
3. Start the new version from the same event log.
4. Rebuild or catch up its projection before dispatching reconciliation commands.

The new version may continue an existing workflow only under the pinned stream version policy; it cannot silently take ownership of the old version's population. If migration is needed, a deliberate adoption event and human-approved policy are required later. There is never a period in which two controller versions can process the same issue population.

## Failure, idempotency, and recovery behavior

### Duplicate intake

- Same delivery retries with the same `commandId`: `findByCommand` returns the original receipt with `replayed: true`; no second event batch or issue is created.
- Different delivery with the same `request.fingerprint` within the same Project and active controller version: atomically append `DuplicateIntakeObserved` to the original stream and return `outcome: "duplicate"`.
- A fingerprint match across Projects is not a duplicate; Project isolation is part of identity.
- A changed request body with an accidentally reused command ID is rejected as an idempotency-key conflict, never silently treated as the old request.

### Controller restart

The controller restarts by replaying the event log or catching up the projection checkpoint. It scans for durable intent events without a corresponding result event and dispatches reconciliation commands using the intent's stable key. In-memory queues are only an optimization; the event log is the work queue and source of truth. A restart cannot lose an accepted intake or forget that an issue was linked.

### Worker stopping between transitions

This first slice does not launch Workers, but its boundary makes the later case precise. A Worker transition must first append its transition-intent event and only then perform an external action; the completed action appends a result event. If the Worker stops after the intent, the restarted controller sees an unresolved intent and retries/reconciles it. If it stops after the result append, replay observes the result and does not repeat the transition. Scope approval remains a prerequisite to any Worker command, and Delivery approval remains a separate high-risk gate; neither can be inferred from a Worker result.

### Partial writes and crashes

- A command batch is all-or-nothing. The storage transaction writes event envelopes and the command receipt together, so there is no valid state with only half of a batch.
- A crash before append commits: retrying the same command ID appends once.
- A crash after intent append but before the adapter call: reconciliation sees the intent and calls the adapter.
- A crash after the adapter succeeds but before the result append: reconciliation calls `ensureIssue` again with the same provider idempotency key, observes the existing issue, then appends `IssueLinked` once.
- A crash after result append but before projection catch-up: status may be temporarily behind, but `observedThrough` exposes the lag; replay catches it up without changing the log.
- A storage failure never causes an adapter result to be fabricated. The command remains unresolved and visible for retry.

The adapter contracts must provide provider-side idempotency or deterministic lookup by `requestKey`/`checkKey`. Without that seam, at-least-once event reconciliation could create duplicate GitHub issues; this is an implementation blocker, not a reason to weaken the log invariant.

### Read model and rebuild

`projection` is a pure fold from `EventEnvelope[]` to `StatusView`, with a materialized index keyed by `WorkflowRef`. The index may be dropped and rebuilt from the log. It is never authoritative and is not accepted as a command precondition without checking its sequence against the log. A status response is read-only and carries `observedThrough`; callers that need a post-command observation can wait for the receipt's `lastSequence` to be projected.

## Verification evidence (design-only)

No production code or validation command is part of this candidate. The following are implementation-time observable scenarios and the evidence they must produce:

| Scenario | Setup/action | Required observable evidence |
|---|---|---|
| Intake acceptance | Dispatch one `AcceptIntake` | One `IntakeAccepted` event, one workflow reference, status phase `intake` |
| Duplicate delivery | Dispatch the same command ID twice | Same receipt on retry; event count unchanged on second call |
| Duplicate request | Dispatch a new command with the same Project/fingerprint | One `DuplicateIntakeObserved`; no second issue workflow |
| Issue retry | Stop after `IssueLinkRequested`; restart and reconcile | One GitHub issue by provider key; exactly one `IssueLinked` result |
| Ownership retry | Stop after `OwnershipCheckRequested`; restart and reconcile | One ownership observation and one result event; no direct status mutation |
| Controller restart | Rebuild projection from the event log | Same `StatusView` and sequence as before restart |
| Partial batch | Inject storage failure during a multi-event append | No half-batch visible; command retry is safe |
| Stale controller | Dispatch an old-version command after handoff | No append; explicit inactive-version rejection |
| Read-only status | Call `readStatus` repeatedly | No new events, adapter calls, or status writes |
| Gate safety | Attempt a future Worker command without Scope approval | Rejected with no Worker transition event; Delivery gate remains independent for high-risk work |
| Learning isolation | Audit runs and propose a policy change | Proposal appears only in Learning stream; active Project policy remains unchanged until approval/release |

The event sequence, append receipts, provider idempotency keys, projection sequence, and controller-fence decisions are the evidence set. A successful HTTP response alone is not evidence of durable work.

## First slice versus later capabilities

**First slice in this candidate:**

- one Project profile, one repository, and one Intake source;
- normalized Work request acceptance and duplicate handling;
- one GitHub issue ensure operation;
- ownership check with explicit evidence or rejection;
- controller-version-fenced durable event stream;
- atomic append and command idempotency;
- restart-safe intent/result reconciliation;
- projection-backed, read-only status;
- recorded two-gate autonomy policy, without starting Workers.

**Later capabilities, deliberately not designed here:**

- Discovery, Triage, Scope approval UI, and approved-scope calculation;
- Worker assignment/execution, repository changes, Pull requests, review, risk checks, merge, and Delivery approval execution;
- child issues, multiple intake providers, multiple repositories per profile, or cross-Project orchestration;
- event archival/compaction, multi-region replication, or event-schema migration tooling;
- automatic policy changes, self-approval, or a production feedback loop.

The Learning workflow is separate rather than a later command in the main run: it reads completed event streams and evidence, creates a versioned Learning proposal in its own namespace, tests it against prior runs, and requires human approval before producing a Policy release. The main controller consumes only an approved pinned release; it cannot append policy changes to its own workflow stream or approve its own proposal.

## Rationale

An append-only event log puts the durable boundary at the point where intent becomes recoverable. The projection is disposable, read-only, and explicit about freshness. This is structurally different from an aggregate/command-centric design: the primary model is an ordered fact stream plus a reducer, not a mutable workflow object whose methods own current state. External actions are represented as intent and result facts, so restart and partial progress are ordinary replay cases instead of exceptional aggregate mutations.

The public command gateway is deep because it hides sequence management, command receipts, controller fencing, adapter retry policy, and projection invalidation behind two operations. The domain event vocabulary is the source of truth for status; no second mutable status machine can drift. The event schema is a domain boundary, so GitHub, intake, and storage representations do not leak into callers.

### Red-flag screen

- **Shallow module:** rejected the design of a `WorkflowLog` with `append`, `load`, `save`, `ensureIssue`, `checkOwner`, and `status` methods that merely forward calls. `dispatch` owns dedupe, sequencing, fencing, and reconciliation; `readStatus` owns projection freshness.
- **Information leakage:** provider payloads and storage rows remain behind adapters/event-log storage. Callers see branded domain references and normalized facts, not GitHub SDK objects or database schemas.
- **Temporal decomposition:** rejected separate `IntakeLoader`, `Validator`, `Transformer`, and `Persister` modules. Commands are grouped by the decision they protect, while the event log and reducer own durable facts and derived state respectively.
- **Pass-through methods:** adapter calls are made by command handlers that append intent/result facts and choose idempotency keys. No layer forwards the same provider-shaped arguments merely to add a name.

### Tradeoffs accepted

- We accept an eventually consistent status projection in exchange for a rebuildable read model and a single durable source of truth; freshness is exposed by `observedThrough`.
- We accept more event records (intent plus result) in exchange for explicit crash recovery and observable external work.
- We accept append-only storage growth in exchange for auditability and deterministic replay; archival is later work.
- We accept at-least-once reconciliation in exchange for simpler restart behavior, requiring provider idempotency keys at the adapter seam.
- We accept that schema evolution needs versioned event handling in exchange for keeping old workflow evidence immutable.
- We accept a separate controller-fence stream in exchange for making version separation enforceable rather than convention-based.

## Rejected alternatives

### Mutable workflow aggregate with command methods

A `WorkflowRun` object with `acceptIntake()`, `ensureIssue()`, `checkOwnership()`, and `start()` would make current state convenient, but it hides the durable intent/result boundary inside mutable methods. Restart recovery, audit history, and projection rebuild become secondary concerns, while callers or the aggregate must coordinate external side effects and persistence. It is not structurally equivalent to this event-log candidate and was rejected for this slice.

### GitHub issue as the source of truth

Writing labels/comments to GitHub and deriving status from them appears simple, but it leaks provider state into workflow policy, makes duplicate/restart semantics dependent on remote search, and cannot represent an intent that has not yet reached GitHub. It also gives a controller version no private durable fence. GitHub remains an adapter, not the event store.

### Durable task queue plus mutable status table

A queue would reliably schedule adapter calls, but a separate mutable status table would lose the complete transition history and make partial writes between queue acknowledgement and status updates difficult to prove safe. It exposes queue acknowledgement and status synchronization to operators. A queue may be an optimization over unresolved intent events later, but it is not the primary boundary here.

## Open questions and risks

- Which durable store provides an atomic event-batch plus command-receipt transaction and a unique fingerprint index per Project?
- What exact provider idempotency or lookup guarantees can the GitHub adapter provide for issue creation after a controller crash?
- Is ownership policy evaluated locally from the Project profile, by a service adapter, or by a human review, and what evidence reference is durable enough?
- Should the first implementation synchronously catch up the projection before returning a dispatch receipt, or should callers use an explicit `observedThrough >= receipt.lastSequence` wait?
- What is the approved procedure and event vocabulary for adopting an existing stream when a controller version is intentionally migrated?
- How long must raw intake bodies and external evidence remain available for replay, and which fields must be redacted before event append?
- Which human authority approves Learning proposals and Policy releases, and how is the active release pinned during controller handoff?

## Next implementation step

Implement the domain event schemas, atomic `AppendOnlyEventLog`, and pure `StatusView` reducer together with replay fixtures for the acceptance, duplicate, restart, partial-write, and stale-controller traces above.
