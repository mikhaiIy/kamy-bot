# Matt run: questions, answers, and assumptions

- Starting commit: `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`
- Run: Matt side of Experiment 001.
- Scope: design only; no production implementation, GitHub mutation, or production worker launch.

## Questions and answers

No blocking clarification question was required. The brief and `CONTEXT.md` define the first slice, the long-term workflow, the ownership responsibilities, the approval concepts, and the separate Learning workflow.

| Question | Answer used by this run | Basis |
| --- | --- | --- |
| What is in the first slice? | Intake source → Work request → GitHub issue → ownership check → durable Workflow run → read-only status. | Experiment 001 fixed task and domain context. |
| What may a Worker decide? | A Worker executes assigned Approved scope and reports Evidence; it does not decide product direction, ownership, Approved scope, or merge authorization. | Experiment brief and domain context. |
| What is the autonomy shape? | Gate 1 is authority/ownership before a run may proceed. Gate 2 is explicit Scope approval before any later implementation action; high-risk delivery approval remains a later gate. | Two-gate requirement plus `CONTEXT.md` approval definitions. |
| How does Learning relate to this design? | Learning is a separate workflow that proposes, tests, and promotes or rolls back policy or capability-pack releases; it does not redesign the main workflow by default. | Domain context and experiment brief. |

## Assumptions recorded for design comparison

1. One intake event identifies one Work request. The first slice creates one GitHub issue for one Work request.
2. A Project profile resolves one configured repository, one ownership policy, one Intake source, and the relevant provider adapters. Project identity is part of every durable run identity.
3. Ownership is checked before a durable run is allowed to advance. A denied or indeterminate check is a terminal recorded outcome for the first slice, not an implicit bypass.
4. Duplicate intake is detected by a stable Intake source identity within a Project. The first accepted identity maps to one GitHub issue and one Workflow run; retries return the existing result rather than creating another issue or run.
5. External calls are not assumed atomic with durable state. The design therefore records intent, result, and recovery state explicitly and makes reconciliation a controlled operation rather than silently retrying an unknown mutation.
6. A Controller restart may replay a pending transition, but a transition is applied only when its expected predecessor and idempotency key still match. A stale Controller version cannot advance a run owned by another active version.
7. Partial writes are observable failures. No design treats an absent or ambiguous write as success.
8. Read-only status is a projection of durable state and evidence. It does not trigger intake, ownership checks, GitHub mutation, or workflow advancement.
9. The exact ownership policy, GitHub issue body/schema, durable store technology, lease/claim duration, and recovery operator are unresolved implementation decisions.
10. The first slice defines extension seams for later Discovery, Triage, implementation, review, risk check, delivery approval, and merge without specifying their internal state machines.

## Proposed vocabulary changes

None are applied to canonical `CONTEXT.md`. Any terminology used by the Matt artifacts is proposal-only and remains under `docs/experiments/mattp/` until the comparison is approved.
