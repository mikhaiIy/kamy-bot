# Pstack interrogate verdict

**Baseline commit:** `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`

## Intent

The selected proposal should provide the safest, clearest first-slice foundation: normalize one intake source into one Work request, ensure exactly one GitHub issue, check and durably record ownership, recover safely across duplicates/restarts/partial writes/between-transition stops, fence controller versions, expose read-only status, preserve two-gate autonomy, and stop before Workers. It must leave later factory phases and the separate Learning workflow extensible without designing them now.

## Reviewers

- Reviewer A: coordinator review lane for correctness and recovery, 3 findings.
- Reviewer B: coordinator review lane for boundary and concurrency, 4 findings.
- Reviewer C: coordinator review lane for autonomy and scope, 4 findings.

The originally spawned OMP reviewer tasks did not produce files at their assigned paths and were stopped under the recovery instruction. The three stable review artifacts were then written from independent adversarial passes over the selected proposal and the same context.

## Act On

### 1. Make controller fencing and cross-version work claims atomic
**Raised by:** A and C. **Category:** critical.

`active()` plus a version partition is not enough to prove that a stale controller cannot mutate GitHub during a handoff. Add a Project-level controller fence with a monotonic epoch and a stable WorkRequest claim independent of version partitions. Require the claim/epoch in every external operation and define the handoff race and durable commit check before implementation.

### 2. Make provider idempotency a testable adapter contract
**Raised by:** A and B. **Category:** critical.

The service must not assume that a provider honors an arbitrary key. Specify deterministic lookup/create behavior, explicit unknown-outcome handling, and evidence that a timeout after remote creation resolves to one issue. Otherwise at-least-once reconciliation can duplicate the shared GitHub record.

### 3. Derive and validate canonical intake identity and digest
**Raised by:** B. **Category:** critical.

The current sketch lets envelope identity, request identity, and digest be caller-supplied and inconsistent. Normalize at the boundary, derive the canonical digest, and reject mismatches before claim. Persist that canonical value for duplicate/collision checks.

### 4. Complete the result and evidence algebra
**Raised by:** A and B. **Category:** critical.

The prose promises identity collision, retryable failure, and human recovery outcomes, but `AdmitResult` does not expose them. Add explicit domain variants and a closed evidence state that distinguishes not-called, unknown-result, and committed-result operations. Define source acknowledgement for each outcome.

## Consider

- **Status consistency:** B found no read-after-command or projection freshness guarantee. Choose authoritative snapshot reads for the first store and specify whether a minimum revision is supported. Do not introduce an eventually consistent projection without a freshness contract.
- **Future Human decision provenance:** C recommends a typed HumanDecision/ApprovedScope value object for the later approval command. Keep Scope `required` in this slice; do not add speculative approval fields now.
- **Future Worker extension seam:** C recommends a separate later checkpoint for Worker action contracts. The first implementation should expose no Worker-capable operation.
- **Provider-specific source literal:** B notes that `sourceKind: "obsidian-daily-note"` couples the domain type to the first adapter. This is a maintainability concern; it does not block the one-source experiment if the normalization seam is explicit.

## Noted

- The event-log candidate's replay and `observedThrough` ideas are useful evidence patterns but are not required as a public first-slice contract.
- Snapshot/CAS persistence remains proportionate if its evidence record and recovery contract are explicit.

## Dismissed

- A public event stream is not required for Learning. CONTEXT.md defines Learning as a separate workflow, and a read-only views/evidence port preserves the boundary without forcing event sourcing into this slice.
- Speculative Worker, Discovery, Triage, or merge types are not missing implementation; they are explicit later capabilities.

## Agreement map

A and C independently identified the missing enforceability of controller-version handoff. A and B independently identified that provider idempotency needs a real contract, not a parameter name. B alone identified canonical identity/digest and status consistency; both are concrete contract gaps. C alone focused on Human decision provenance and future Worker boundaries; those are extension risks rather than first-slice blockers. No reviewer found that the aggregate should be replaced by the event-log architecture for this slice.

## Verdict

The aggregate boundary remains the selected shape, but implementation is blocked until the four Act On contract gaps are incorporated into the design. The review found no reason to switch to event-log-first persistence. Verification evidence must include the new fence epoch, stable cross-version claim, canonical digest, explicit outcome/evidence variants, and provider-side idempotency trace.
