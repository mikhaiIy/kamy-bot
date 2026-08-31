# Matt candidate evaluation

- Starting commit: `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`
- Scope: Matt-side design comparison only. No production implementation, GitHub mutation, or production Worker launch.
- Inputs: `candidate-state-transition.md`, `candidate-event-log.md`, `questions-assumptions.md`, and `scoring-rubric.md`.

## Structural difference

| Candidate | Authoritative durable model | Status model | Main strength | Main cost |
| --- | --- | --- | --- | --- |
| Transactional state transition | Current run snapshot plus explicit transition/evidence records; transition history is audit, not replay authority | Read current snapshot | Small operational contract with direct status and local recovery | Must keep snapshot and transition records atomically consistent |
| Append-only event log | Ordered per-run domain event stream | Deterministic projection, rebuildable from stream | Strong replay/audit semantics and explicit immutable history | Event schema, projection lag/rebuild, and migration machinery arrive in the first slice |

Both candidates use explicit commands, durable idempotency receipts, Project and Controller-version scoping, an external GitHub intent/result protocol, explicit Human authority, blocked ambiguity, and Interface-level behavioral verification. They differ in what is authoritative and in how much replay machinery is mandatory.

## Scores

Scores use the shared 1–5 rubric. Safety and failure coverage are gating concerns; simplicity is not allowed to erase recovery guarantees.

| Criterion | Transactional state transition | Append-only event log | Evidence |
| --- | ---: | ---: | --- |
| Domain clarity | 5 | 5 | Both distinguish Work request, GitHub issue, run, state/events, evidence, Project, Controller version, and Human decision. |
| Interface depth and locality | 5 | 5 | Both place legality, idempotency, authority, version fencing, and recovery behind a small engine Interface. |
| Failure-case coverage | 5 | 5 | Both cover duplicates, restart timing, non-atomic GitHub effects, partial writes, Worker stop, stale commands, and version separation. |
| Simplicity | 4 | 2 | Snapshot status is direct; event log additionally requires projection rebuild/freshness and event-schema lifecycle. |
| Testability | 5 | 5 | Both make Adapter faults and command-boundary crashes observable through the Interface. Event prefixes make replay especially direct. |
| Agent executability | 4 | 4 | Explicit commands and gates bound actions in both; the event candidate adds stream/projection concepts agents must understand. |
| Operational safety | 5 | 5 | Both block ambiguity, require authorization, fence Controller versions, and prevent duplicate issue creation by marker reconciliation. |
| **Total (informational)** | **33** | **31** | Totals do not override safety gates. |

## Decision

Select the **transactional state-transition candidate** for the next Kamy-Bot task, subject to approval. It provides the smallest useful deep Module for the fixed slice while retaining the safety machinery that matters: atomic local transitions, durable intent before GitHub, idempotent commands, compare-and-set revisions, explicit blocked recovery, Human authority, Project isolation, and Controller-version fencing.

The selected design should incorporate two ideas from the event-log candidate without making event sourcing mandatory:

1. Keep immutable transition and evidence records with command identity, revision, actor, and payload digest. They provide auditability and replay fixtures without making a projection rebuild the only way to know current state.
2. Treat external intent, verified result, and ambiguous outcome as distinct durable facts. A deterministic read model may be rebuilt or checked from transition records, but the current state snapshot remains authoritative for status and command preconditions.

## Rejected alternatives from the comparison

- **Append-only event log as primary authority:** rejected for the next slice because projection freshness, event schema evolution, rebuild behavior, and migration policy add first-slice complexity not required to establish the contract. Retain immutable transition/evidence records and replay fixtures as selected ideas.
- **GitHub as the Workflow state machine:** rejected by both candidates because GitHub cannot provide durable local atomicity, Project/version fencing, command receipts, or explicit pre-call intent.
- **In-memory Controller state:** rejected because restart boundaries and duplicate external effects become unsafe.
- **Generic CRUD repository exposed to callers:** rejected because it is a shallow Module; callers would repeat transition legality, idempotency, authorization, and recovery rules.
- **Generic event bus without per-run ordering:** rejected because ordering, deduplication, expected revision, and status semantics leak into subscribers.
- **Synchronous GitHub call inside a long durable transaction:** rejected because it cannot create distributed atomicity and holds local resources across provider latency.
- **Framework-first design:** rejected because an agentic framework is an Implementation or Adapter choice behind Kamy-Bot Interfaces, not the Workflow definition.
- **Unscoped global Controller scanner:** rejected because it permits Project and Controller-version population overlap.

## Remaining risks

1. The chosen snapshot and transition record must commit atomically; otherwise status can disagree with its audit trail.
2. GitHub marker design must avoid leaking sensitive Intake content while remaining deterministic for reconciliation.
3. Controller lease fencing and cutover rules need a concrete operational contract before production.
4. Ownership policy and Human actor authentication remain product and security decisions, not defaults to infer in implementation.
5. The read-only status compatibility and freshness guarantee must be explicit before consumers depend on it.

## Verification conclusion

The candidate comparison is a design plan, not executed test evidence. Before implementation, the selected proposal must be checked through deterministic Interface-level scenarios for golden path, exact and conflicting duplicate intake, every restart boundary, partial local writes, GitHub success-before-result-commit, ambiguous GitHub outcomes, Worker stop between transitions, stale commands, unauthorized Human decisions, cross-version mutation, and read-only non-mutation. Production implementation is intentionally out of scope for this run.
