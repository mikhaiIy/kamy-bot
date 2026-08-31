# Experiment 001: workflow-engine design

## Purpose

Compare the Matt Pocock and pstack development workflows before Kamy-Bot implementation begins.

## Platform context

Kamy-Bot is a new, reusable AI software factory.

The long-term flow is:

```text
Intake source
→ Work request
→ GitHub issue
→ Discovery
→ Triage
→ Human approval
→ Worker implementation through SSSF or ADW
→ Pull request
→ Integration review
→ Required Human or visual approval
→ Merge
```

Kamy-Bot supports multiple Projects through Project profiles and Adapters. Each Project has its own repository, ownership policy, workflow labels, and external integrations.

Workers execute Approved scope. Workers do not decide product direction, ownership, Approved scope, or merge authorization.

Kamy Jemput is a separate stable orchestrator. This experiment must not change Kamy Jemput.

## Question

Which development workflow produces the clearest and safest design for the first Kamy-Bot foundation slice?

## Fixed task

Design this foundation slice without implementing production code:

```text
Intake source
→ GitHub issue
→ ownership check
→ durable Workflow run
→ read-only status
```

The slice must support later Discovery, Triage, SSSF or ADW implementation, review, and merge phases. It must not design or implement those later phases in detail.

The design must cover one Project profile, one GitHub repository, one Intake source, duplicate intake, a Controller restart, a Worker that stops between two state transitions, and separation between two Controller versions.

## Two independent runs

Run each workflow from the same baseline commit. Record the exact commit before either run starts.

Create two separate worktrees or branches:

```text
experiment/mattp-workflow-engine
experiment/pstack-workflow-engine
```

Store outputs in separate directories:

```text
docs/experiments/mattp/
docs/experiments/pstack/
```

Do not let either run read the other run's proposal before scoring.

### Matt run

Use the repository-local Matt skills:

- `grill-with-docs`
- `domain-modeling`
- `codebase-design`

Use OMP task agents when the workflow needs independent design candidates.

Produce at least two structurally different architecture candidates. Produce one selected proposal and one synthesis note.

### pstack run

Use the repository-local pstack skills:

- `architect`
- `arena`
- `interrogate`

Use OMP task agents. Do not use Cursor-only model rules, cloud agents, Graphite, or `gt`.

Produce at least two structurally different architecture candidates. Produce one selected proposal and one synthesis note.

## Required proposal contents

Each run must produce:

1. The domain terms and their relationships.
2. The smallest useful Workflow state model.
3. The main interface and its invariants.
4. The Adapter seams for Intake source, GitHub, and durable state.
5. Duplicate, restart, partial-write, and Controller version behavior.
6. A verification plan using observable behavior.
7. Rejected alternatives and the reason for rejection.
8. A clear separation between first-slice requirements and later factory capabilities.

## Scoring

Score each selected proposal from 1 to 5 for:

- Domain clarity.
- Interface depth and locality.
- Failure-case coverage.
- Simplicity.
- Testability.
- Agent executability.
- Operational safety.

Do not choose a result by line count or visual polish.

## Comparison result

After both runs, record:

- The selected ideas from the Matt run.
- The selected ideas from the pstack run.
- Ideas rejected by each run.
- Important disagreements.
- Which workflow should be used for the next Kamy-Bot task.
- Which pstack skills, if any, should be ported into OMP.
- Which unresolved decisions block production implementation.

## Non-goals

- Do not implement production code.
- Do not choose a production database or transport only because it is familiar.
- Do not change Kamy Jemput.
- Do not install a second live controller.
- Do not let either workflow change GitHub issues or launch production workers.

No production implementation begins until the comparison result is recorded and approved.
