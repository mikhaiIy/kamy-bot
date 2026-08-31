# Matt run: scoring rubric

- Starting commit: `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`
- Purpose: compare structurally different first-slice designs without using line count or visual polish.

## Criteria and anchors

Score each candidate from 1 to 5:

| Criterion | 1 | 3 | 5 |
| --- | --- | --- | --- |
| Domain clarity | Terms overlap or hide ownership | Main terms are named but some relationships are implicit | Work request, issue, run, state, evidence, Project, Controller version, and Human decision have distinct responsibilities and explicit relationships |
| Interface depth and locality | Callers repeat workflow rules | Some rules are centralized; important policy leaks | A small Interface hides transition legality, idempotency, recovery, authority, and version isolation; changes stay local |
| Failure-case coverage | Happy path only | Some failures are described but recovery is vague | Duplicate, restart timing, partial local write, ambiguous external effect, Worker stop, stale command, and version separation have observable outcomes |
| Simplicity | Complexity is broad or speculative | Reasonable core with some extra machinery | Smallest useful state and Interface for the fixed slice; complexity is justified by a named safety invariant |
| Testability | Requires provider or process luck | Fakes can cover the main path | Interface-level deterministic scenarios inject races, crashes, ambiguity, stale revisions, and read-only assertions |
| Agent executability | Agents can infer scope or authority | Boundaries exist but some actions are ambiguous | Commands, allowed transitions, authority, evidence, and stop/retry behavior are explicit and bounded |
| Operational safety | Duplicate or unauthorized mutation is plausible | Safety depends on operator discipline | Durable ownership/version fencing, idempotency, explicit human gates, and blocked ambiguity prevent unsafe mutation |

## Decision rule

The selected proposal must maximize operational safety and failure-case coverage first, then prefer domain clarity, deep Interface/locality, testability, agent executability, and simplicity. A lower simplicity score is acceptable when the added machinery directly handles GitHub's non-transactional seam or crash recovery. No candidate is selected by total score alone: any candidate with an unsafe duplicate, authority, Project isolation, or Controller-version story is rejected regardless of total.
