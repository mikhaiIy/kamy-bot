# Selected pstack proposal: deep WorkflowRun command boundary

**Baseline commit:** `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`

## Usage (caller's view)

The first-slice caller uses one `WorkflowService`. It submits a normalized intake envelope, reconciles recoverable runs after restart, and reads a domain status view. It does not construct GitHub requests, manipulate storage, coordinate temporal stages, or start workers.

```ts
const workflow = createWorkflowService({
  profiles,
  intake: obsidianDailyNotes,
  github,
  runs,
  clock,
});

const receipt = await workflow.admit({
  projectId: "kamy-bot",
  controllerVersion: "controller-2026-08-31.1",
  intake: await obsidianDailyNotes.next(),
});

if (receipt.kind === "accepted" || receipt.kind === "duplicate" ||
    (receipt.kind === "rejected" && receipt.reason === "not-owned")) {
  await obsidianDailyNotes.acknowledge(receipt.intakeIdentity);
}
```

A controller restart uses the same command boundary:

```ts
const recovered = await workflow.reconcile({
  projectId: profile.id,
  controllerVersion: controller.version,
});
```

A status endpoint can only read:

```ts
const view = await workflow.status({
  projectId: profile.id,
  controllerVersion: controller.version,
  runId,
});
```

`status` has no external side effects. The source acknowledges only a durable acceptance, duplicate, or terminal ownership rejection. A malformed envelope is not claimed.

## Problem

The slice crosses an intake provider, GitHub, an ownership policy, and durable state. The safety decision is one admission decision, but each external call can succeed before the process stops. Duplicate intake and two controller versions must not create or process two records for the same issue population. The design must also stop before Workers and preserve both human autonomy gates. The selected shape concentrates these invariants in a deep command service around one durable `WorkflowRun` aggregate.

## Domain relationships

- `ProjectProfile` identifies one repository, ownership policy, intake source, workflow labels, and exactly one active `ControllerVersion`.
- `IntakeSource` supplies identity and request content. It does not decide scope.
- `WorkRequest` is the normalized user-originated unit of work.
- One Work request maps to one GitHub issue and one Workflow run in this experiment.
- `WorkflowRun` is one durable attempt. It owns state, operation keys, ownership evidence, gate facts, and recovery metadata.
- `Worker` is a later actor. It cannot start until explicit Human Scope approval.
- `LearningWorkflow` is a separate read-only consumer of run views and evidence. It proposes policy or capability-pack changes but cannot mutate runs or approve itself.

Immutable identity is `(ProjectId, IntakeIdentity)`, with the active controller version also pinned to the run. A version handoff transfers the Project's issue-population claim explicitly; a cached or stale controller cannot process it.

## Smallest useful state model

```text
Intake --claim--> issue-pending --issue linked--> ownership-pending
                                              |                  |
                                      not owned              owned
                                              v                  v
                                    ownership-rejected   awaiting-scope-approval
```

`issue-pending` and `ownership-pending` are recovery states. `awaiting-scope-approval` is the end of this slice. No state in this proposal starts a Worker or marks Scope approved.

State invariants:

1. `issue-pending` has no issue reference; every later state has exactly one.
2. `ownership-pending` has an issue and no ownership result.
3. `ownership-rejected` records immutable negative evidence and cannot start a Worker.
4. `awaiting-scope-approval` records immutable positive ownership evidence and a cleared admission gate, but Scope remains `required`.
5. Every committed transition increments a durable revision and records an operation key.
6. A retryable failure leaves the semantic state unchanged and records failure metadata.
7. A malformed input or identity collision is rejected before claiming or overwriting a run.

## Public contract sketch

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

type WorkflowState =
  | "issue-pending"
  | "ownership-pending"
  | "ownership-rejected"
  | "awaiting-scope-approval";

type WorkflowRunView = Readonly<{
  runId: RunId;
  projectId: ProjectId;
  controllerVersion: ControllerVersion;
  intakeIdentity: IntakeIdentity;
  issue: { id: IssueId; number: number; url: string } | null;
  state: WorkflowState;
  ownership: "owned" | "not-owned" | null;
  gates: {
    admission: "not-cleared" | "cleared";
    scope: "required" | "approved";
    delivery: "not-applicable" | "required" | "approved";
  };
  lastFailure: { retryable: boolean; reasonCode: string } | null;
  revision: number;
  updatedAt: Instant;
}>;

type AdmitResult =
  | { kind: "accepted"; intakeIdentity: IntakeIdentity; run: WorkflowRunView }
  | { kind: "duplicate"; intakeIdentity: IntakeIdentity; run: WorkflowRunView }
  | { kind: "rejected"; reason: "not-owned"; intakeIdentity: IntakeIdentity; run: WorkflowRunView }
  | { kind: "rejected"; reason: "invalid-intake"; intakeIdentity?: IntakeIdentity }
  | { kind: "controller-conflict"; active: ControllerVersion };

interface WorkflowService {
  admit(input: {
    projectId: ProjectId;
    controllerVersion: ControllerVersion;
    intake: IntakeEnvelope;
  }): Promise<AdmitResult>;

  reconcile(input: {
    projectId: ProjectId;
    controllerVersion: ControllerVersion;
  }): Promise<{ runs: readonly WorkflowRunView[]; skipped: number }>;

  status(input: {
    projectId: ProjectId;
    controllerVersion: ControllerVersion;
    runId: RunId;
  }): Promise<WorkflowRunView | null>;
}
```

The three methods are the complete external interface. There is no public `load`, `save`, `transition`, `createIssue`, `checkOwnership`, or `appendEvent` method. Callers request an intent; the service owns sequencing and returns a durable outcome.

## Internal shape and adapter seams

The aggregate is the sole owner of mutable run state. Its internal transitions are not caller contracts:

```ts
class WorkflowRun {
  static start(input: StartRun): WorkflowRun;
  ensureIssue(issue: IssueRef): Transition;
  recordOwnership(result: Ownership): Transition;
  recordFailure(failure: Failure): Transition;
  view(): WorkflowRunView;
}
```

Each transition carries `operationId`, `expectedRevision`, and the next state. Durable commit is compare-and-swap plus operation-key deduplication. Intent and external-result evidence are retained in the run's durable evidence record so recovery can distinguish “not called,” “called with unknown result,” and “result committed.” This grafts the event-log candidate's strongest recovery idea without making an event stream the public or primary model.

```ts
interface ProjectProfiles {
  active(projectId: ProjectId): Promise<ProjectProfile>;
}

interface GitHubAdapter {
  ensureIssue(input: {
    repository: string;
    request: WorkRequest;
    idempotencyKey: string;
  }): Promise<IssueRef>;
  checkOwnership(input: {
    repository: string;
    issue: IssueRef;
    ownershipPolicy: string;
    operationKey: string;
  }): Promise<OwnershipCheck>;
}

interface DurableRunStore {
  claimOrFind(input: ClaimInput): Promise<ClaimResult>;
  load(runId: RunId, partition: RunPartition): Promise<WorkflowRunSnapshot | null>;
  listRecoverable(partition: RunPartition): Promise<readonly WorkflowRunSnapshot[]>;
  commit(input: CommitInput): Promise<"committed" | "already-committed" | "revision-conflict">;
}
```

`claimOrFind` atomically enforces `(ProjectId, IntakeIdentity)` uniqueness and the active controller claim. `commit` atomically records the new revision, operation key, state, and evidence. GitHub issue ensuring must use a stable Project plus intake key and discover an issue created before a timeout. Wire payloads, database rows, event envelopes, and framework objects stay private to adapters.

## Module map

```text
workflow/WorkflowService
  ├─ profile lookup and controller fence
  ├─ WorkflowRun transition policy
  ├─ issue and ownership orchestration
  ├─ operation-key and CAS reconciliation
  ├─ durable recovery
  └─ domain status projection
       ├─ ProjectProfiles adapter
       ├─ GitHubAdapter
       ├─ DurableRunStore
       └─ Clock
intake/obsidian-adapter -> IntakeEnvelope + post-success acknowledgement
http/status -> WorkflowService.status only
learning/auditor -> read-only views/evidence; no command access
```

Modules are grouped by knowledge, not by `load → validate → transform → save` timing. The service is deep: it hides the external coordination and policy while exposing only three operations.

## Required failure behavior

- **Duplicate intake:** atomically claim by Project plus intake identity. Same digest returns the original run as `duplicate`; a changed digest returns an identity collision and never overwrites. A different delivery with the same request digest is also a duplicate. GitHub is not called again unless the existing run is recoverable.
- **Issue timeout after remote creation:** remain `issue-pending` with retryable failure. Retry with the same provider key; adapter lookup returns the existing issue, never a second issue.
- **Durable write failure after issue creation:** return no success receipt. On retry or restart, the durable claim and deterministic provider key converge on the existing issue.
- **Ownership unavailable:** remain `ownership-pending` with retryable evidence. Unknown is never treated as owned.
- **Controller restart:** `reconcile` lists only the Project/version partition's recoverable snapshots. It replays the next safe intent using the operation key. No in-memory cursor is authoritative.
- **Stop between transitions:** either the old revision is durable and the idempotent operation is repeated, or the new revision is durable and retry is a no-op. A Worker is not launched by this slice; future Worker actions must use the same command/commit boundary.
- **Controller-version conflict:** every command carries the version. A stale version is rejected before external mutation. State and claims are isolated by Project and version; handoff is an explicit authorized operation, never silent adoption.
- **Read-only status:** status reads durable state only and never creates issues, checks ownership, writes storage, or starts Workers.

## Two-gate autonomy and Learning separation

Ownership/admission is not a Human approval. After positive ownership, the Scope gate is still `required` and must later receive an explicit Human decision covering Approved scope, repository paths, acceptance conditions, and risk. Delivery approval is a separate gate required only for high-risk changes after later review/risk work. This slice records both gate policies but implements neither later phase.

The Learning workflow reads run views and evidence through a separate read-only interface. It may create a versioned Learning proposal, test it against prior runs, and seek Human approval for a Policy release. It cannot mutate a run, approve Scope or Delivery, activate a controller, or publish its own proposal. The main architecture is not redesigned around Learning.

## Verification plan

The implementation checkpoint must use observable behavior at the `WorkflowService` seam:

1. Submit one intake twice: observe one run, one issue, and `accepted` then `duplicate`.
2. Reuse the identity with a changed digest: observe collision and no overwrite.
3. Inject timeout after issue creation, restart, and reconcile: observe one issue and eventual ownership-pending or awaiting-scope-approval.
4. Inject durable commit failure after issue creation: restart and observe deterministic issue reuse.
5. Stop between each transition: reconcile and observe monotonic revisions with no duplicate ownership result.
6. Run concurrent admits: observe one claim winner and one duplicate without conflicting request content.
7. Run stale and active controller versions: observe stale rejection before GitHub mutation and isolated partitions.
8. Call status repeatedly: observe no external calls or durable writes and only domain fields.
9. Expose Learning only the read interface: observe views/proposals without run or policy mutation.
10. Attempt future Worker execution without Scope approval: observe rejection and no worker transition.

Retain adapter traces, operation keys, durable revisions, controller-fence decisions, returned receipts, and restart outcomes. These are behavioral evidence, not claims about private class structure.

## First slice and later capabilities

**First slice:** one Project profile, one repository, one intake source; normalized Work request; idempotent GitHub issue; ownership evidence; durable WorkflowRun; duplicate/restart/partial-write recovery; controller fencing; read-only status; stop before Worker execution.

**Later:** Discovery, Triage, Scope approval command, Worker execution, Pull Requests, review, risk checks, Delivery approval and merge, additional providers/projects, child issues, rich event replay, Learning proposals and releases, and controller handoff tooling.

## Next implementation step

Build an in-memory WorkflowService and fake adapters against this contract, then exercise the listed failure traces before choosing the production durable store. No production implementation begins before this design checkpoint is approved.
## Interrogate result and required pre-implementation amendments

The adversarial review is recorded in `interrogate-verdict.md` and reviewer lanes A–C. The aggregate shape remains selected, but implementation MUST first close these contract gaps:

1. Add a Project-level controller fence with a monotonic epoch and a stable WorkRequest claim independent of version-partitioned run snapshots. An external operation carries the claimed epoch; handoff and commit behavior are atomic and explicitly tested.
2. Define `GitHubAdapter.ensureIssue` as deterministic lookup/create with provider-side idempotency, plus an explicit unknown-outcome result that blocks unsafe automatic retries when the guarantee is unavailable.
3. Normalize intake at the boundary. Derive the canonical WorkRequest identity and digest from source identity and canonical request content; reject envelope/request mismatches before claiming.
4. Complete the domain result and evidence unions with identity collision, retryable failure, human recovery, and not-called/unknown-result/committed-result distinctions. Source acknowledgement must be defined for every result.
5. State the first-slice status consistency guarantee. Prefer an authoritative snapshot read and a documented revision comparison; do not add an eventually consistent projection without a freshness contract.

These amendments strengthen enforceability without adding event sourcing, Worker types, or later workflow phases to the first slice.
