# Interrogate reviewer A: correctness and recovery

**Baseline commit:** `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`

## Findings

### 1. [critical] Controller fencing is not defined as one atomic invariant
**Location:** `selected-proposal.md`, “Internal shape and adapter seams” and “Required failure behavior”

**Finding:** The proposal requires stale controllers to be rejected before external mutation, but its shown operations do not define a fencing token/epoch that survives an active-version handoff.

**Evidence:** `ProjectProfiles.active` reads the active version; `claimOrFind` claims by Project plus intake identity and says it enforces the active controller claim; then the service can call `GitHubAdapter.ensureIssue`. A handoff can commit between the read/claim and that call. The stale process can therefore still mutate GitHub even though line 240 promises rejection “before external mutation.” Partitioning state by version does not prevent two versions from touching one Project issue population.

**Suggestion:** Make the controller claim an atomic, leased fencing record with a monotonically increasing epoch. Require every external operation to carry the epoch and require the durable boundary to validate the claim immediately before dispatch; define the handoff race and whether an in-flight external call is allowed to finish. Keep one Project-wide intake/issue uniqueness index independent of version partitions.

### 2. [critical] Provider idempotency is a requirement, not a complete adapter contract
**Location:** `selected-proposal.md`, `GitHubAdapter.ensureIssue` and failure behavior

**Finding:** The design depends on exactly-once issue identity after an unknown result, but the adapter interface only accepts an `idempotencyKey` and returns an `IssueRef`.

**Evidence:** If GitHub creates an issue and the process dies before `commit`, retrying is safe only if the adapter can atomically discover the prior issue by that key. The proposal says the adapter “must” do this but supplies no lookup result, duplicate detection contract, or explicit unresolvable-outcome state. A provider that ignores the key would create a second issue on reconciliation.

**Suggestion:** Specify a provider contract with deterministic lookup/create semantics and an explicit `unknown-outcome` result that blocks automatic retry when the guarantee is unavailable. Add an acceptance scenario that proves the adapter trace, not only the service call count.

### 3. [warning] Recovery evidence is named but not typed
**Location:** `selected-proposal.md`, line describing “intent and external-result evidence”

**Finding:** The selected proposal grafts intent/result facts but does not show their domain type or legal transitions.

**Evidence:** Recovery must distinguish “not called,” “called with unknown result,” and “result committed.” Without an explicit evidence state, an implementation can collapse timeout, provider success, and durable commit failure into `lastFailure`, causing reconciliation to repeat or skip the wrong external action.

**Suggestion:** Add a closed evidence state to the sketch, with operation key, attempt, provider observation, and durable commit relation. Define which states are retryable and which require human recovery.
