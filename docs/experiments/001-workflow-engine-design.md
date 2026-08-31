# Experiment 001: Agentic Development Workflow design

## Purpose

Compare the Matt Pocock and pstack development workflows before Kamy-Bot implementation begins.

## Platform context

Kamy-Bot is a new, reusable Agentic Development Workflow platform.

An Agentic Development Workflow is a software delivery process in which agents perform bounded development actions, produce evidence, and pass work through explicit workflow states.

Kamy-Bot is an agentic software factory. Its long-term Agentic Development Workflow is:

```text
Intake source
→ Work request
→ GitHub issue
→ Discovery
→ Triage
→ Scope approval
→ Agent implementation
→ Pull request
→ Integration review
→ Risk check
→ Delivery approval when required
→ Merge
```

Kamy-Bot supports multiple Projects through Project profiles and Adapters. Each Project has its own repository, ownership policy, workflow labels, and external integrations.

Kamy-Bot will define its own Workflow. It is not required to follow an existing software factory implementation or framework.

The Agentic framework is an implementation choice behind Kamy-Bot interfaces. This experiment must not select a framework only because another system uses it.

Workers execute Approved scope. Workers do not decide product direction, ownership, Approved scope, or merge authorization.

Kamy Jemput is a separate stable orchestrator. This experiment must not change Kamy Jemput.

The Workflow requires Scope approval before implementation. It requires Delivery approval only for high-risk changes. Low-risk changes can merge after all required checks pass.

Kamy-Bot will improve its Workflow policy and Agent capability packs through a separate Learning workflow. The Learning workflow examines prior runs, creates a Learning proposal, tests the proposal, and promotes or rolls back an approved Policy release. It does not redesign the main architecture by default.

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

The slice must support later Discovery, Triage, agent implementation, review, and merge phases. It must not design or implement those later phases in detail.

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

## Automatic clarification protocol

The coordinator answers agent questions automatically.

- Agents may ask at most five blocking questions.
- Agents must ask only questions that the context does not answer.
- The coordinator answers from `CONTEXT.md`, this brief, or a conservative default.
- The coordinator does not invent a product or safety decision.
- If a question requires a Human decision, the coordinator records it as unresolved.
- The coordinator gives the same answer to both runs when the question applies to both.
- The coordinator records each answer in `docs/experiments/clarifications.md`.

The two runs must use the same clarification record before final scoring. The clarification record has one writer: the coordinator.

## Herdr execution

Run Matt and pstack in separate new Herdr sessions. Each session uses OMP task agents. Neither session may write production code, change GitHub, or launch production workers.

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
- Do not choose an external workflow framework only because it is familiar.
- Do not change Kamy Jemput.
- Do not install a second live controller.
- Do not let either workflow change GitHub issues or launch production workers.

No production implementation begins until the comparison result is recorded and approved.
