# Candidate: WorkflowRun aggregate and command boundary

**Baseline:** `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`

This is a design-only candidate for Experiment 001's first slice:

```text
Intake source → Work request → GitHub issue → ownership check
→ durable Workflow run → read-only status
```

It supports one Project profile, one repository, and one Intake source in the first implementation. The interfaces retain Project and adapter identity so later Projects do not require a different domain model.

## Usage (caller's view)

The caller depends on one `WorkflowService`. It submits a normalized Intake record, asks the service to reconcile recoverable runs after a restart, and reads a status view. It does not load an aggregate, validate transitions, construct GitHub requests, write storage, or acknowledge an Intake source before durable success. A terminal ownership rejection is durable and may be acknowledged; a malformed envelope is not claimed and should be dead-lettered by the source.

```ts
const workflow = createWorkflowService({
  profiles,
  intake: obsidianDailyNotes,
  github: githubAdapter,
  runs: durableRunStore,
  clock,
});

const receipt = await workflow.admit({
  projectId: "kamy-bot",
  controllerVersion: "controller-2026-08-31.1",
  intake: await obsidianDailyNotes.next(),
});

// A source acknowledges only after admit has durably accepted or recorded the
// item. A duplicate or terminal ownership rejection is also safe to acknowledge.
if (
  receipt.kind === "accepted" ||
  receipt.kind === "duplicate" ||
  (receipt.kind === "rejected" && receipt.reason === "not-owned")
) {
  const identity =
    receipt.kind === "rejected"
      ? receipt.intakeIdentity
      : receipt.run.intakeIdentity;
  await obsidianDailyNotes.acknowledge(identity);
}
```

### Call site: Intake poller

```ts
for await (const item of obsidianDailyNotes.items()) {
  const result = await workflow.admit({
    projectId: profile.id,
    controllerVersion: controller.version,
    intake: item,
  });

  if (
    result.kind === "accepted" ||
    result.kind === "duplicate" ||
    (result.kind === "rejected" && result.reason === "not-owned")
  ) {
    await obsidianDailyNotes.acknowledge(item.identity);
  }
}
```

The adapter maps a daily-note entry to `IntakeEnvelope`; the service owns the idempotency key and all domain decisions. A source retry after a lost response cannot create a second issue.

### Call site: Controller restart or a stopped Worker

```ts
const recovered = await workflow.reconcile({
  projectId: profile.id,
  controllerVersion: controller.version,
});

for (const run of recovered.runs) {
  logger.info({ runId: run.runId, state: run.state }, "workflow run recovered");
}
```

`reconcile` scans only the durable partition for the active Project and controller version. It advances each recoverable aggregate through its next safe command. A process or Worker stopping after an external call but before the next durable transition is therefore ordinary recovery, not a special caller protocol.

### Call site: Read-only status endpoint

```ts
app.get("/workflow-runs/:runId", async (request, response) => {
  const view = await workflow.status({
    projectId: profile.id,
    controllerVersion: controller.version,
    runId: request.params.runId,
  });

  response.status(view ? 200 : 404).send(view);
});
```

The endpoint receives a domain `WorkflowRunView`, not a GitHub SDK object or a storage row. `status` never creates an issue, checks ownership, changes a label, or starts a Worker.


## Shape

### Caller-facing types

The following are domain types, not transport or persistence types. Branded identifiers prevent accidental mixing of a GitHub issue number, an Intake identity, a Project, and a Controller version.

```ts
type ProjectId = string & { readonly __brand: "ProjectId" };
type ControllerVersion = string & { readonly __brand: "ControllerVersion" };
type IntakeIdentity = string & { readonly __brand: "IntakeIdentity" };
type RunId = string & { readonly __brand: "RunId" };
type IssueId = string & { readonly __brand: "IssueId" };
type Instant = string & { readonly __brand: "Instant" };

type WorkRequest = Readonly<{
  identity: IntakeIdentity;
  title: string;
  body: string;
  digest: string;
}>;

type IntakeEnvelope = Readonly<{
  identity: IntakeIdentity;
  request: WorkRequest;
  sourceKind: "obsidian-daily-note";
  receivedAt: Instant;
}>;

type AdmitInput = Readonly<{
  projectId: ProjectId;
  controllerVersion: ControllerVersion;
  intake: IntakeEnvelope;
}>;

type ReconcileInput = Readonly<{
  projectId: ProjectId;
  controllerVersion: ControllerVersion;
}>;

type StatusInput = Readonly<{
  projectId: ProjectId;
  controllerVersion: ControllerVersion;
  runId: RunId;
}>;

type WorkflowState =
  | "issue-pending"
  | "ownership-pending"
  | "ownership-rejected"
  | "awaiting-scope-approval";

type Ownership =
  | { readonly kind: "owned"; readonly checkedAt: Instant; readonly evidence: string }
  | { readonly kind: "not-owned"; readonly checkedAt: Instant; readonly evidence: string };

type WorkflowRunView = Readonly<{
  runId: RunId;
  projectId: ProjectId;
  controllerVersion: ControllerVersion;
  intakeIdentity: IntakeIdentity;
  issue: { readonly id: IssueId; readonly number: number; readonly url: string } | null;
  state: WorkflowState;
  ownership: Ownership | null;
  gates: Readonly<{
    readonly admission: "not-cleared" | "cleared";
    readonly scope: "required" | "approved";
    readonly delivery: "not-applicable" | "required" | "approved";
  }>;
  lastFailure: { readonly retryable: boolean; readonly message: string } | null;
  revision: number;
  updatedAt: Instant;
}>;
// Private support types used by the aggregate implementation and adapters.
type IssueRef = Readonly<{
  id: IssueId;
  number: number;
  url: string;
}>;

type ProjectProfile = Readonly<{
  id: ProjectId;
  repository: string;
  intakeSource: "obsidian-daily-note";
  ownershipPolicy: string;
  activeControllerVersion: ControllerVersion;
}>;

type GateState = Readonly<{
  admission: "not-cleared" | "cleared";
  scope: "required" | "approved";
  delivery: "not-applicable" | "required" | "approved";
}>;

type Failure = Readonly<{
  retryable: boolean;
  message: string;
}>;

type StartRun = Readonly<{
  runId: RunId;
  projectId: ProjectId;
  controllerVersion: ControllerVersion;
  intakeIdentity: IntakeIdentity;
  requestDigest: string;
  startedAt: Instant;
}>;

type EnsureIssueInput = Readonly<{
  projectId: ProjectId;
  repository: string;
  request: WorkRequest;
  idempotencyKey: string;
}>;

type CheckOwnershipInput = Readonly<{
  projectId: ProjectId;
  repository: string;
  issue: IssueRef;
  ownershipPolicy: string;
}>;

type OwnershipCheck =
  | Ownership
  | { readonly kind: "unknown"; readonly reason: string };

type ClaimInput = Readonly<{
  partition: RunPartition;
  intake: IntakeEnvelope;
  requestDigest: string;
}>;

type ClaimResult =
  | { readonly kind: "new"; readonly snapshot: WorkflowRunSnapshot }
  | { readonly kind: "existing"; readonly snapshot: WorkflowRunSnapshot }
  | { readonly kind: "controller-conflict"; readonly active: ControllerVersion };

type WorkflowRunSnapshot = RunState;

type CommitInput = Readonly<{
  partition: RunPartition;
  transition: Transition;
}>;


type AdmitResult =
  | { readonly kind: "accepted"; readonly run: WorkflowRunView }
  | { readonly kind: "duplicate"; readonly run: WorkflowRunView }
  | {
      readonly kind: "rejected";
      readonly reason: "not-owned";
      readonly intakeIdentity: IntakeIdentity;
      readonly run: WorkflowRunView;
    }
  | { readonly kind: "rejected"; readonly reason: "invalid-intake" }
  | { readonly kind: "controller-conflict"; readonly active: ControllerVersion };

type ReconcileResult = Readonly<{
  runs: readonly WorkflowRunView[];
  skipped: number;
}>;

interface WorkflowService {
  /** Idempotently admits one Intake envelope and reaches the next safe state. */
  admit(input: AdmitInput): Promise<AdmitResult>;

  /** Recovers and advances pending runs for one Project/controller partition. */
  reconcile(input: ReconcileInput): Promise<ReconcileResult>;

  /** Returns a read-only domain view; it has no external side effects. */
  status(input: StatusInput): Promise<WorkflowRunView | null>;
}
```

### Signatures and aggregate command model

The three methods are the complete external interface. `admit` and `reconcile` are commands even though they return views. There is no public `load`, `save`, `transition`, `checkOwnership`, `createIssue`, `appendEvent`, or `acknowledge` method. Callers cannot accidentally perform half of a Workflow operation.


#### WorkflowRun aggregate internals

`WorkflowRun` is the only owner of its mutable state. These methods are private to the Workflow implementation and are shown to make the command boundary concrete; they are not caller contracts.

```ts
class WorkflowRun {
  private constructor(private state: RunState) {}

  static start(input: StartRun): WorkflowRun;
  ensureIssue(issue: IssueRef): Transition;
  recordOwnership(result: Ownership): Transition;
  markScopeGatePending(): Transition;
  retryableFailure(failure: Failure): Transition;
  view(): WorkflowRunView;
}

type RunState = Readonly<{
  runId: RunId;
  projectId: ProjectId;
  controllerVersion: ControllerVersion;
  intakeIdentity: IntakeIdentity;
  requestDigest: string;
  issue: IssueRef | null;
  ownership: Ownership | null;
  state: WorkflowState;
  gates: GateState;
  revision: number;
  lastFailure: Failure | null;
}>;

type Transition = Readonly<{
  operationId: string;
  expectedRevision: number;
  next: RunState;
}>;
```

The aggregate rejects impossible transitions internally, for example recording ownership without an issue, changing an already recorded ownership result, or clearing the scope gate from this slice. A repeated command with the same operation key returns the previously committed result. A command whose expected revision loses a compare-and-swap is reloaded and evaluated again inside the service.

## Module map

```text
intake/obsidian-adapter
  └─ emits IntakeEnvelope and acknowledges only after WorkflowService.admit succeeds

workflow/WorkflowService (external seam)
  ├─ profile admission and active-controller claim
  ├─ WorkflowRun aggregate command handling
  ├─ GitHub issue and ownership orchestration
  ├─ idempotency and optimistic revision retry
  ├─ durable commit/recovery policy
  └─ domain WorkflowRunView projection
       ├─ profiles/ProjectProfiles adapter
       ├─ github/GitHubAdapter
       ├─ state/DurableRunStore
       └─ time/Clock

read/status endpoint
  └─ calls WorkflowService.status only

learning/auditor (later, separate seam)
  └─ consumes read-only run views/evidence; cannot call admit or reconcile
```

This is intentionally not a chain of `IntakeService → IssueService → OwnershipService → RunRepository`. Those objects would expose temporal stages and force callers to coordinate them. The service has internal seams for testing and adapters, but one command boundary owns the decision.



## Domain relationships

- A **Project profile** identifies one Project, its repository, ownership policy, Intake source, workflow labels, and active Controller version.
- An **Intake source** supplies a normalized `IntakeEnvelope` with request content and source identity. It does not decide scope or ownership.
- A **Work request** is the domain identity and content derived from that envelope. Experiment 001 maps one Work request to one GitHub issue.
- A **GitHub issue** is the shared work record. The aggregate stores a domain `IssueRef`; GitHub wire representations remain in the Adapter.
- A **Workflow run** is one durable attempt for one Work request. Its aggregate owns state, transition rules, idempotency, ownership-result recording, and gate status.
- A **Worker** is not started by this slice. Later Workers will be admitted only after the aggregate records Scope approval.
- **Human decisions** are represented as gate facts, not inferred from an issue label. Scope approval is mandatory before implementation; Delivery approval is required only for high-risk changes.
- A **Learning workflow** reads run views/evidence through a separate read-only port. It has no command access to this aggregate and cannot change policy or start work.

Relationship invariants:

```text
ProjectProfile 1 ── active ControllerVersion 1
ProjectProfile 1 ── Intake source 1
WorkRequest 1 ── GitHub issue 1 ── WorkflowRun 1
WorkflowRun 1 ── immutable (ProjectId, ControllerVersion, IntakeIdentity)
```

The GitHub issue uniqueness rule is global to the Project and Intake identity even though run snapshots are partitioned by controller version. A version handoff must transfer the durable claim before a new version can process that issue population.

## State model

The smallest useful durable state model is:

```text
                 ensure issue
Intake ─────────────────────────→ issue-pending
                                     │
                          issue exists│
                                     ▼
                              ownership-pending
                              │              │
                  not owned ──┘              └── owned
                       ▼                           ▼
                ownership-rejected       awaiting-scope-approval
```

`Intake` is the external source state, not an acknowledged run state. The durable aggregate is created/claimed before attempting external work so that a crash has a place to record the next command. The semantic chain is still issue creation, ownership decision, then a run awaiting approval; `issue-pending` and `ownership-pending` are recovery states, not approval.

State invariants:

1. `issue-pending` has no `IssueRef`; every other state has exactly one.
2. `ownership-pending` has an issue and no ownership result.
3. `ownership-rejected` has an immutable `not-owned` result and can never start a Worker.
4. `awaiting-scope-approval` has an immutable `owned` result and admission gate `cleared`.
5. Scope approval is not synthesized from ownership. This slice cannot set `scope: approved`.
6. Every committed transition increments `revision` and records an operation key; no transition mutates a prior revision.
7. A retryable failure leaves the run in its prior semantic state with `lastFailure`; a non-retryable validation failure is returned without claiming or overwriting a run.

Status is deliberately a view of the aggregate snapshot. It may report a transient recovery state and the latest failure, but it never exposes storage revisions as a protocol requirement or GitHub transport fields.

## Adapter seams

Adapters are explicit dependencies of the implementation, never domain types in the public interface.

```ts
interface ProjectProfiles {
  active(projectId: ProjectId): Promise<ProjectProfile>;
}

interface GitHubAdapter {
  ensureIssue(input: EnsureIssueInput): Promise<IssueRef>;
  checkOwnership(input: CheckOwnershipInput): Promise<OwnershipCheck>;
}

interface DurableRunStore {
  claimOrFind(input: ClaimInput): Promise<ClaimResult>;
  load(runId: RunId, partition: RunPartition): Promise<WorkflowRunSnapshot | null>;
  listRecoverable(partition: RunPartition): Promise<readonly WorkflowRunSnapshot[]>;
  commit(input: CommitInput): Promise<"committed" | "already-committed" | "revision-conflict">;
}

interface Clock {
  now(): Instant;
}

type RunPartition = Readonly<{
  projectId: ProjectId;
  controllerVersion: ControllerVersion;
}>;
```

`GitHubAdapter.ensureIssue` must use the stable Project + Intake identity as its idempotency key and must return the same `IssueRef` after a timeout/retry. `DurableRunStore.claimOrFind` atomically enforces the Intake uniqueness claim and controller lease. Its storage schema, transaction mechanism, serialization, and indexes stay private. The composition root wires one concrete adapter for the first slice; the seams are justified because provider and storage variation are real, not because every function needs an interface.
## Command behavior and failure/idempotency rules

### Admission and duplicate Intake


The idempotency identity is `(ProjectId, IntakeSource identity)`, with a request digest stored on the claim. The same identity and digest return `duplicate` with the original run and issue. The same identity with a different digest returns `invalid-intake`/collision and does not overwrite the original Work request. The source must not be acknowledged before the service returns a durable result.

`admit` first verifies that the source belongs to the Project profile and that the supplied Controller version is active. It then atomically claims or finds the intake identity. A duplicate never calls GitHub again unless the existing aggregate is in a recoverable state; even then `ensureIssue` uses the same key.

### GitHub partial failure

- If issue creation times out, the aggregate remains `issue-pending` with a retryable failure. Reconciliation repeats `ensureIssue` with the same key; the adapter must discover an issue created before the timeout rather than creating another.
- If GitHub created the issue but the durable commit fails, the caller receives failure. A later admission/reconciliation finds the claim and deterministically reuses that issue. There is no "create another issue to be safe" path.
- If an issue is deleted or the adapter reports a non-retryable mismatch, the run records a failure requiring exceptional human recovery; the service does not silently replace the shared work record.

### Ownership result

Ownership checking is performed only after an `IssueRef` exists. `owned` clears the admission gate and leaves the run at `awaiting-scope-approval`. `not-owned` records evidence and terminates admission at `ownership-rejected`. An unavailable or ambiguous check remains `ownership-pending`; unknown is never treated as owned. Repeated checks are safe because the aggregate accepts the first committed result and reconciliation returns it without changing it.

### Durable persistence and CAS

Each aggregate command is committed with `expectedRevision` and an operation ID in one durable transaction. If the commit returns `revision-conflict`, the service reloads that run, checks whether the operation already committed, and otherwise applies the command to the newer state. If the write is unavailable, no success receipt is returned. A process may die after the external call and before commit or after commit and before response; both cases converge through the idempotency key and revision check.

### Controller restart and Worker stop

On restart, the controller calls `reconcile` for its Project/version partition. The service lists only pending runs and executes their next safe operation; callers do not replay arbitrary transitions. A Worker/process stopping between two transitions leaves either the old state (repeat the idempotent external operation) or the new committed state (no-op on retry). No in-memory cursor is authoritative.

The first slice does not run implementation Workers, but the same rule is a required contract for later Worker actions: a Worker reports one command intent to the aggregate, and the aggregate commits the resulting transition or returns a retryable failure. The Worker cannot advance a run by editing GitHub or storage directly.

### Controller-version separation

Every command includes `ControllerVersion`. The profile adapter exposes exactly one active version for a Project. The service rejects stale versions with `controller-conflict` before external mutation. Durable state is partitioned by `(ProjectId, ControllerVersion)`, and each run stores the version immutably.

A version handoff is an explicit operational action outside this slice: stop the old version, transfer the Project's durable issue-population claim, then activate the new version. Until that claim transfer commits, the new version receives a conflict. This prevents two versions from processing one issue population while preserving isolated run state. A new version cannot silently create a second issue for an Intake identity held by an old version.

## Two-gate autonomy

There are two separate autonomy gates; ownership/admission is a prerequisite, not a substitute for either human gate:

1. **Scope gate (every implementation):** after ownership is recorded, the run waits for an explicit Human Scope approval covering the Work request, Approved scope, repository paths, acceptance conditions, and risk. No Worker may start while this gate is required.
2. **Delivery gate (high risk only):** after later implementation/review/risk checks, a high-risk run requires explicit Human Delivery approval before merge. Low-risk runs do not require this second approval once all required checks pass.

This candidate records the first gate as `required` and intentionally has no command that approves it. It has a delivery-gate field so later phases do not accidentally treat absence of a second approval as permission, but it does not design implementation, review, risk, or merge transitions.

## Learning workflow separation

The Learning workflow is a separate read-only consumer of `WorkflowRunView` and later evidence. It may identify patterns and create a versioned Learning proposal, but it cannot invoke `admit` or `reconcile`, mutate a run, approve a gate, change the active Controller version, or publish a Policy release. Policy releases are reviewed and approved outside this aggregate and are not inferred from run state. This first slice therefore keeps the main command boundary stable while allowing later auditing without event-log or policy machinery in the aggregate.

## Why this is structurally different from the other candidate

This candidate treats the `WorkflowRun` snapshot as the unit of ownership and the command as the unit of change. A caller asks the aggregate to admit or reconcile; transition legality, idempotency, ownership recording, controller claims, and persistence happen together behind one deep seam. There is no public event stream, event-handler pipeline, replay contract, or event-log-as-domain-model. An event-log-centric candidate would make facts/events and their append/replay semantics the primary seam; this candidate deliberately makes current run invariants and command outcomes primary. The distinction is structural, not a different naming convention.

## Red-flag screen

- **Shallow module:** rejected. The public interface has three operations, while it hides external issue idempotency, ownership interpretation, state-machine validation, CAS retries, controller claims, and recovery. `status` is a genuine read operation, not a forwarding method.
- **Information leakage:** rejected. GitHub SDK requests/responses, storage rows, event schemas, and adapter cursors do not cross the seam. Callers receive domain identifiers and a stable view.
- **Temporal decomposition:** avoided. Intake polling, issue creation, ownership checking, and persistence occur at different times but remain inside the aggregate service because they protect one admission decision. Public methods express intent (`admit`, `reconcile`), not `load → validate → create → save` stages.
- **Pass-through methods:** avoided. `WorkflowService` owns orchestration and policy; the adapters each translate a real external system. There is no public repository wrapper or one-method-per-transition facade.

## Problem

The first slice crosses three external systems while needing one coherent safety decision: should this particular Work request be accepted into this Project's Workflow? The difficult cases are not the happy path; they are duplicate Intake, an issue created before a process dies, a Worker stopping between transitions, and an old Controller version waking up after a handoff. This candidate puts those decisions inside a deep `WorkflowRun` aggregate behind one command-oriented service. Durable snapshots, operation keys, and controller claims are implementation details of that service. Intake, GitHub, and durable state remain explicit adapters at private seams. The public seam exposes only admission, reconciliation, and a read-only view.

## Rationale

The aggregate is deep because callers learn only how to submit an Intake envelope, recover a Project/version partition, and read a domain view. The implementation absorbs the hard coordination: external idempotency, partial writes, optimistic concurrency, ownership evidence, and controller-version fencing. The aggregate's state is current and directly queryable, which makes the first slice operationally simple without forcing every caller to understand event ordering or build a projection.

This shape also gives locality. A correction to duplicate semantics, ownership evidence, or stale-controller behavior is made once in the command implementation. Tests cross the same `WorkflowService` seam as callers and can use in-memory adapters for deterministic failure scenarios. The adapters remain replaceable because Intake, GitHub, and durable storage are provider-specific; the domain does not depend on their wire protocols.

The boundary is intentionally narrower than a general workflow engine. It stops at `awaiting-scope-approval`, does not start a Worker, and does not pretend that an issue label is a Human decision. Later states can extend the aggregate only after their invariants and approval rules are specified.

## Tradeoffs accepted

- We accept a richer aggregate implementation in exchange for a small command interface and one place to enforce every run invariant.
- We accept snapshot/CAS persistence in exchange for direct read status and a simpler first recovery path; historical evidence must be added deliberately rather than assumed to exist as an event log.
- We accept serial command handling per run in exchange for deterministic duplicate and restart behavior. Parallelism across independent runs can be added inside reconciliation without sharing a mutable aggregate.
- We accept an explicit operational controller handoff in exchange for never allowing two Controller versions to process one issue population.
- We accept a recovery state before a durable run is ready in exchange for not losing an issue or Work request when a process stops between external and durable operations.
- We accept that ownership evidence is immutable for this run; exceptional re-check or policy changes require an explicit recovery operation rather than silently changing history.
- We accept that this first slice does not provide a general audit/event replay API. Learning and richer evidence are later capabilities with their own read seam.

## Rejected alternatives

### Public pipeline of stage services

A caller-facing `IntakeService → IssueService → OwnershipService → RunRepository` pipeline was rejected. It is temporally decomposed and shallow: every caller would need to know ordering, retry each boundary, correlate IDs, and decide what to do after a partial write. It leaks GitHub and storage representations and makes duplicate handling a convention instead of an invariant.

### Event-log-first WorkflowRun

An event-log-first design with public `append(event)`, replay, and projection APIs was rejected for this first slice. It would make event schema, ordering, and projection freshness part of every caller's interface before the product needs replay or multi-consumer history. It can be a later internal persistence strategy, but the command boundary should remain the source of domain decisions.

### Stateless controller plus database procedures

A thin controller that sends SQL procedures or GitHub calls directly was rejected. It hides little behind a large operational surface, makes ownership and controller-version rules difficult to test as one invariant, and permits a stale process to bypass the intended command path. The aggregate service can still use transactions internally without exposing them.

### One global Project workflow singleton

A singleton run registry shared by all Projects and Controller versions was rejected. It would conflate ownership policies and repositories, make duplicate keys ambiguous, and violate the requirement that run state be isolated per Project/version. Partitioning is explicit in every command and storage operation.

## Open questions and risks

- Does the production durable adapter provide an atomic claim, issue-idempotency lookup, and compare-and-swap commit, or must the first implementation choose a store that does?
- What exact evidence establishes ownership for each Project profile, and which Human role can authorize exceptional recovery from `ownership-rejected`?
- What is the operational protocol and authorization for transferring an issue-population claim during a Controller-version handoff?
- Should a permanently unavailable GitHub issue move to a separate `needs-human-recovery` state, or is `lastFailure.retryable: false` sufficient for the first implementation?
- Is source acknowledgement at-least-once with a durable cursor, or can an Intake adapter provide an atomic acknowledge-plus-submit operation?
- When later Scope approval is added, should approval be a command on this aggregate or a separate Human-decision port that the aggregate validates and records?
- Which run evidence must be retained for the Learning workflow, and what retention period is required if snapshots alone are insufficient?
- How should status represent a run whose Controller version is no longer active: readable historical state, or a distinct `superseded` view?

## Verification plan and evidence

This artifact is design-only; no production code, GitHub mutation, Worker launch, formatter, linter, build, or project-wide test is required. The implementation should prove the contract through behavior at the `WorkflowService` seam:

1. Submit the same Intake identity twice and observe one `IssueRef`, one run, and an `accepted` then `duplicate` result. Submit the same identity with a different digest and observe no overwrite.
2. Inject an `ensureIssue` timeout after the adapter has created an issue. Reconcile twice and observe one issue and eventual `ownership-pending` or `awaiting-scope-approval`, never a second issue.
3. Inject a durable commit failure after issue creation and restart the controller. Observe recovery from the durable claim and reuse of the deterministic issue key.
4. Inject a Worker/process stop between each pair of transitions. Restart and reconcile; observe convergence, monotonic revisions, and no duplicate ownership result.
5. Run two concurrent admits for one identity and observe one claim winner, one duplicate result, and no lost or conflicting Work request.
6. Run an old and active Controller version against the same Project. Observe stale-version rejection and no external mutation by the old version; observe that state partitions remain isolated after an authorized handoff.
7. Return `ownership-rejected` and verify that no Scope gate is cleared and no Worker-capable command is emitted.
8. Read status repeatedly and verify it is side-effect free, returns domain fields only, and reflects durable recovery state.
9. Give a Learning auditor only the read port and verify it can inspect views but cannot admit, reconcile, approve, or alter policy.

The evidence to retain for implementation is the adapter trace (idempotency key and external result), durable revision/operation records, returned domain receipts, and restart/reconciliation outcomes. Those observations test behavior rather than private class structure.

## First slice versus later capabilities

### First slice

- One Project profile, one repository, and one Intake source.
- Normalize one Intake envelope into one Work request.
- Idempotently ensure one GitHub issue.
- Check and durably record ownership.
- Persist the WorkflowRun aggregate with the four states above.
- Fence stale Controller versions and isolate Project/version state.
- Recover duplicate, restart, partial-write, and between-transition failures.
- Expose read-only status.
- Stop at `awaiting-scope-approval`; do not start Workers.

### Later capabilities

- Discovery and Triage, including child issues if the product approves them.
- Human Scope approval command and Approved scope representation.
- Worker execution, Pull Requests, review, risk checks, Delivery approval, and merge.
- Multiple Intake providers and Project profiles beyond the first fixture.
- Rich evidence retention, replay, or an event-log-backed audit projection if required.
- Learning auditor, Learning proposals, policy-release testing, canaries, promotion, and rollback.
- More elaborate Controller handoff tooling and multi-Project scheduling.

Later capabilities must extend the aggregate through new domain commands and invariants; they must not make callers coordinate the internal transition sequence.

## Next implementation step

Build an in-memory `WorkflowService` against the types above, with fake Intake/GitHub/durable adapters that exercise duplicate, timeout, CAS-conflict, restart, and stale-controller scenarios before selecting the production durable adapter.
