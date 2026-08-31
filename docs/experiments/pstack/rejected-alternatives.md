# Pstack rejected alternatives

**Baseline commit:** `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`

These alternatives were considered across both pstack candidates. Rejection is for Experiment 001's first slice, not a permanent ban on every implementation detail.

## Public temporal pipeline

`IntakeService → IssueService → OwnershipService → RunRepository` exposes execution order and forces every caller to correlate IDs, retry boundaries, and partial writes. It is shallow and temporally decomposed. The selected command boundary hides those decisions in one service.

## Event-log-first public model

The event-log candidate scored strongly on recovery evidence, but a public append/replay/projection contract makes event schema, sequence, projection lag, and schema evolution caller concerns before the first slice needs them. Its strongest ideas—stable operation keys, intent/result evidence, atomic batches, and replay-style verification—are grafted internally into the aggregate contract. Public event sourcing remains a later option.

## GitHub as workflow source of truth

Using issue labels, comments, or searches as the workflow state leaks provider representation into policy, cannot represent an intent that has not reached GitHub, and gives controller fencing no durable private boundary. GitHub remains an adapter.

## Queue plus mutable status table

A durable queue can schedule adapter work, but a separate status table creates a synchronization problem between acknowledgement, external side effects, and status writes. It also loses the first-slice history needed to explain uncertain outcomes. An unresolved intent queue may later optimize reconciliation; it is not the domain model here.

## Stateless controller with database procedures

Direct controller calls to SQL and GitHub hide little behind a broad operational surface. Ownership, idempotency, gate, and controller-version rules become scattered procedures and are harder to test as one invariant. A transaction remains allowed inside `DurableRunStore`, but not as the public workflow contract.

## One global singleton run registry

A registry shared by Projects or controller versions makes duplicate keys ambiguous and can mix repositories or ownership policies. It violates state isolation. Project identity, stable WorkRequest claim, controller fence epoch, and version-scoped run state must remain explicit.

## Speculative full factory state machine

Designing Discovery, Triage, Worker execution, review, risk checks, Delivery approval, and merge now would enlarge the state space and invite approval inference. This slice ends at `awaiting-scope-approval`; later phases require their own contracts and verification.
