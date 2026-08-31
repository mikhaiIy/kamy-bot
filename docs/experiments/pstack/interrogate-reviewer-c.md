# Interrogate reviewer C: autonomy and scope

**Baseline commit:** `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`

## Findings

### 1. [critical] The controller cannot prove that the issue-population claim is transferred
**Location:** `selected-proposal.md`, domain relationships and controller-version behavior

**Finding:** The proposal states that a version handoff transfers a Project-wide issue-population claim, but the public and internal contracts show only per-run `claimOrFind` and version partitions.

**Evidence:** A new controller version can receive a Work request with the same Project/intake identity after the old version has a run in its partition unless the store has a stable cross-version uniqueness record. The proposal says that such a record exists conceptually, but it is not in `RunPartition`, `ClaimInput`, or `ClaimResult`. “Explicit authorized operation” is a procedure, not an enforceable contract.

**Suggestion:** Model a Project-level controller fence and WorkRequest claim separately from version-scoped run state. Require the new version to present the transferred epoch and have the store return the existing run or an explicit conflict. Add concurrent handoff/admit evidence.

### 2. [warning] Two-gate policy is recorded but the first-slice contract does not expose authorization provenance
**Location:** `selected-proposal.md`, `WorkflowRunView.gates` and autonomy section

**Finding:** The proposal correctly refuses to infer Scope approval from ownership, but the view has only gate status and no stated authority/evidence shape for a future Human decision.

**Evidence:** Later Workers must use Approved scope and Human decisions are load-bearing domain terms. If later code adds `scope: "approved"` without a decision identifier, approver, timestamp, and approved paths/conditions, the aggregate can satisfy its enum while losing the authorization boundary. This is not needed to run the first slice but is a design risk at the extension seam.

**Suggestion:** Keep the first slice at `scope: required`, but reserve a typed HumanDecision/ApprovedScope value object as the only legal input to a future approval command. Do not add a boolean approval flag.

### 3. [warning] Worker stop behavior is asserted without a concrete future transition boundary
**Location:** `selected-proposal.md`, stop-between-transitions behavior

**Finding:** The proposal covers the scenario in prose but does not define whether a future Worker action is one aggregate command, an external side effect with an intent record, or a separate worker lease.

**Evidence:** “Use the same command/commit boundary” is directionally safe, but later repository mutations and evidence production may have different retry and ownership semantics. Without naming the extension seam, an implementation could bypass the aggregate and edit GitHub/storage directly.

**Suggestion:** State the non-goal clearly and require a later design checkpoint for Worker action contracts. The first implementation should assert that no Worker-capable operation exists, rather than adding speculative Worker types now.

### 4. [dismissed] Event sourcing is required for Learning
**Location:** comparison against event-log candidate

**Finding:** A reviewer might demand a public event stream so Learning can replay runs.

**Evidence:** CONTEXT.md defines Learning as a separate workflow and does not require event sourcing. The selected proposal already exposes read-only views/evidence; public replay would enlarge the first-slice surface and couple the main architecture to a future audit mechanism.

**Decision:** Dismissed for this scope. Preserve evidence sufficient for a later read adapter and revisit replay only if a concrete Learning requirement needs it.
