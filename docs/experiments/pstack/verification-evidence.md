# Pstack verification evidence

**Baseline commit:** `1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f`

## Evidence captured

### Starting point

The baseline command was run before candidate generation:

```text
git rev-parse HEAD
1e42dc116a823ff422f5744c1ddf4b85f3bd1c3f
```

Both candidates and every final pstack artifact record this exact commit.

### Artifact inventory

A repository glob found these pstack-only Markdown artifacts:

```text
arena-cross-judge.md
candidates/aggregate.md
candidates/event-log.md
interrogate-reviewer-a.md
interrogate-reviewer-b.md
interrogate-reviewer-c.md
interrogate-verdict.md
rejected-alternatives.md
scoring-rubric.md
selected-proposal.md
synthesis-note.md
unresolved-decisions.md
verification-evidence.md
```

A baseline-commit search found the exact starting commit in every listed artifact. No artifact was written outside `docs/experiments/pstack/` by this pstack run.

### Design comparison

The two candidate files were read end to end. They are structurally different:

- Aggregate candidate: mutable `WorkflowRun` snapshot and three-operation command boundary.
- Event-log candidate: append-only intent/result facts plus reducer-backed status projection.

The recorded cross-judge scored aggregate 34/35 and event-log 28/35. The aggregate base was selected for first-slice locality and proportionality. Stable operation keys, intent/result evidence, and replay-style failure traces were grafted from the event-log candidate without making event sourcing public.

### Adversarial review

Three independent review lanes were recorded under the interrogate rubric. Consensus identified four pre-implementation contract blockers:

1. controller handoff needs an atomic Project-level fence epoch and cross-version WorkRequest claim;
2. GitHub issue ensuring needs deterministic provider lookup/create and unknown-outcome behavior;
3. intake normalization must derive and validate canonical identity and digest;
4. result and evidence unions must explicitly model collisions, retryable failure, human recovery, and external-operation states.

Status consistency, Human decision provenance, and the future Worker boundary are recorded as considered risks. The selected proposal and synthesis note include the review verdict and required amendments.

### Workspace safety

The verification status command reported only untracked experiment directories:

```text
?? docs/experiments/legacy/
?? docs/experiments/mattp/
?? docs/experiments/pstack/
```

The pstack workflow did not read or modify the legacy or Matt directories. No GitHub operation, production-worker launch, production-code edit, formatter, linter, build, or test command was run. The untracked legacy/Matt directories are reported as workspace state, not attributed to this run.

## Checkpoint result

The pstack side is complete at the design checkpoint. Production implementation is intentionally not started. Approval and resolution of the contract blockers in `unresolved-decisions.md` are required before the next implementation task.
