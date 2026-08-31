# Pstack scoring rubric

**Baseline commit:** `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`

This rubric is the pstack-side scoring instrument for Experiment 001. It applies to both candidates without rewarding line count or visual polish. Each criterion receives 1–5 points.

| Criterion | 1 point | 3 points | 5 points |
|---|---|---|---|
| Domain clarity | Terms or boundaries conflict with CONTEXT.md | Core terms are present but some ownership is implicit | Work request, issue, run, profile, controller, gates, and Learning workflow have explicit relationships and owners |
| Interface depth and locality | Callers coordinate stages or see provider/storage details | Some policy is hidden, but key sequencing leaks | A small public surface hides sequencing, idempotency, policy, and persistence decisions at one boundary |
| Failure-case coverage | Duplicate/restart/partial-write cases are absent or unsafe | Main failures are named but convergence is incomplete | Duplicate intake, external timeout, partial durable write, restart, between-transition stop, and stale controller converge safely |
| Simplicity | First slice is dominated by machinery for later phases | Complexity is justified but some future concerns leak in | Smallest useful model; future phases are named without being designed in detail |
| Testability | Behavior depends on live providers or private implementation | Some seams and scenarios are testable | Domain boundary and adapter seams support deterministic observable scenario/replay checks |
| Agent executability | Worker or caller must infer scope, ownership, or authorization | Execution rules exist but have ambiguous transitions | Approved scope, ownership, state, and evidence boundaries prevent agents from making human decisions |
| Operational safety | Duplicate work or controller overlap is possible | Safety depends partly on procedure | Durable idempotency, ownership checks, human gates, and controller-version fencing are enforceable invariants |

**Selection rule:** prefer the candidate with the stronger total only when it has no critical safety gap. If totals are close, prefer the smaller public surface and more local invariant ownership. Record worthwhile ideas from losing candidates as explicit grafts, not as an unexamined average.
