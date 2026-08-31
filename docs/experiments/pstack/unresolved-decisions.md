# Pstack unresolved decisions

**Baseline commit:** `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`

These decisions remain open at the design checkpoint. The first four are contract blockers identified by interrogate; no production implementation should start while they are implicit.

## Contract blockers

1. **Controller fence:** What durable store and lease/epoch protocol makes a Project-level active-controller claim atomic with admission and safe across handoff races? Must a stale in-flight external call be allowed to finish, and how is its result quarantined?
2. **Cross-version work identity:** What stable Project plus WorkRequest claim/index prevents a new controller version from creating a second run or issue for an identity held by an old version while preserving version-isolated run state?
3. **GitHub unknown outcomes:** What provider-side lookup/create or idempotency guarantee is available after a timeout? If the provider cannot guarantee it, which explicit human-recovery state prevents automatic duplicate creation?
4. **Canonical intake:** Which source identity and canonical request serialization define the WorkRequest digest? The normalizer must reject inconsistent envelope/request identities before durable claim.
5. **Result/evidence algebra:** What exact domain variants represent identity collision, retryable failure, unknown external result, and human recovery? Which results permit source acknowledgement?

## Remaining design choices

6. **Status consistency:** Is the first status read authoritative from the snapshot store, or can it be eventually consistent? If a read-after-command guarantee is required, should `status` accept a minimum revision?
7. **Ownership authority:** Is ownership evaluated locally from Project policy, by a provider adapter, or by a Human decision? What evidence reference is durable and what role can authorize exceptional recovery?
8. **Source acknowledgement:** Is intake at-least-once with a durable source cursor, or can the adapter provide atomic acknowledge-plus-submit? How are malformed envelopes dead-lettered without being claimed?
9. **Human decision contract:** When Scope approval is added, which typed HumanDecision/ApprovedScope value object carries approver, authority, timestamp, paths, acceptance conditions, and risk? Which role can approve high-risk Delivery?
10. **Evidence retention:** Which operation traces and external evidence are retained for the separate Learning workflow, for how long, and with what redaction?
11. **Controller handoff authorization:** Which authorized human or operational control performs handoff, and how is the epoch transfer evidenced and reversible?
12. **Source type generalization:** Should normalized intake use a provider-neutral source identity now, or keep the first Obsidian source literal until a second provider exists?

No question requires an automatic clarification from the supplied context. Human authority, provider guarantees, retention, and deployment protocol remain intentionally unresolved rather than invented.
