# Experiment 001: workflow-engine design

## Question

Which development workflow produces the clearest, safest design for the first Kamy-Bot vertical slice?

## Fixed task

Design this vertical slice without implementing production code:

```text
intake source → GitHub issue → ownership check → durable workflow run → read-only status
```

The design must cover one project profile, one GitHub repository, one intake source, duplicate intake, a controller restart, and a worker that stops between two state transitions.

## Two independent runs

### Matt run

Use the repository-local Matt skills:

- `grill-with-docs`
- `domain-modeling`
- `codebase-design`

Produce an architecture proposal without changing production code.

### pstack run

Use the pstack experiment skills:

- `architect`
- `arena` only if there is a genuine architectural fork
- `interrogate` after selecting a candidate

Produce an architecture proposal without changing production code.

## Isolation

Run each experiment from the same commit, `4432efc`, in a separate branch or worktree:

- `experiment/matt-workflow-engine`
- `experiment/pstack-workflow-engine`

Do not let either run read the other run's proposal before scoring.

## Required proposal contents

Each run must produce:

1. The domain terms and their relationships.
2. The smallest useful workflow state model.
3. The main interface and its invariants.
4. The adapter seams for intake, GitHub, and durable state.
5. Duplicate, restart, and partial-write behavior.
6. A verification plan using observable behavior.
7. Rejected alternatives and the reason for rejection.

## Scoring

Score each proposal from 1 to 5 for:

- Domain clarity.
- Interface depth and locality.
- Failure-case coverage.
- Simplicity.
- Testability.
- Agent executability.
- Operational safety.

Do not choose a winner by line count or visual polish.

## Result

After both runs, record:

- The selected ideas from each run.
- Ideas rejected from each run.
- Which workflow should be used for the next Kamy-Bot task.
- Which pstack skills, if any, should be ported to OMP.

No production implementation begins until the comparison result is recorded.
