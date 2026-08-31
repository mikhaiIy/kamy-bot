# Interrogate reviewer B: boundary and concurrency

**Baseline commit:** `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`

## Findings

### 1. [critical] Intake identity and request digest can disagree
**Location:** `selected-proposal.md`, `WorkRequest`, `IntakeEnvelope`, and duplicate behavior

**Finding:** The caller supplies both identity fields and the digest. The contract does not require that the envelope identity equals the request identity or that the digest is computed from canonical request content.

**Evidence:** An envelope can carry identity A and a request carrying identity B. A caller can also reuse an identity with a different body and an old digest. The prose promises collision detection, but the type sketch cannot tell an implementation which value is authoritative or whether to reject mismatches. This can split duplicate claims or treat changed content as the original request.

**Suggestion:** Normalize and validate at the intake boundary: derive the Work request identity from the source identity, compute a canonical digest internally, and reject an envelope when supplied identity/digest values do not match. Store the canonical digest in the durable claim.

### 2. [critical] Public result types omit promised collision and retry outcomes
**Location:** `selected-proposal.md`, `AdmitResult` and failure behavior

**Finding:** The result algebra is incomplete for the behavior required by the prose.

**Evidence:** The duplicate section promises an identity collision result, and the recovery section promises retryable failures and no success receipt after an uncertain durable write. `AdmitResult` has only `invalid-intake`, `not-owned`, and `controller-conflict`; it does not distinguish identity collision, retryable/unresolved admission, or human recovery. An implementation would need untyped exceptions or hidden conventions, so callers cannot acknowledge or retry safely from the contract.

**Suggestion:** Add explicit result variants for `identity-conflict`, `retryable-failure`, and `needs-human-recovery`, each carrying only the domain evidence required for caller action. Define source acknowledgement for each variant.

### 3. [warning] Status consistency and visibility during recovery are unspecified
**Location:** `selected-proposal.md`, `status` contract and verification plan

**Finding:** The design says status reads durable state but does not state whether it is linearizable, can be stale, or how a caller observes a just-committed transition.

**Evidence:** The proposal grafts event-shaped evidence and asks verification to observe recovery, but `status` returns only a view and has no revision/“observed through” contract beyond a numeric revision. During a restart or after a successful command response, a read could return the prior state and clients have no defined convergence rule.

**Suggestion:** State the consistency guarantee. For the first snapshot store, require status to read the committed revision from the same authoritative partition and let callers request/compare `minRevision` if read-after-command visibility is needed. Do not add an eventually consistent projection unless the freshness contract is explicit.

### 4. [nit] The proposal uses a provider-specific source literal in the domain type
**Location:** `selected-proposal.md`, `IntakeEnvelope.sourceKind`

**Finding:** `sourceKind: "obsidian-daily-note"` is acceptable for one source but makes the domain type provider-specific while the domain glossary calls it an adapter concern.

**Evidence:** The proposal otherwise keeps provider wire types private and claims future provider variation. Adding a second source would require changing the domain type rather than adding an adapter.

**Suggestion:** Use a branded source identity/type behind a normalized envelope, while keeping the first profile restricted to the Obsidian adapter. This is a maintainability improvement, not a first-slice safety blocker.
