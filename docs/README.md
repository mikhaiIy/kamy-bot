# Kamy-Bot documentation

This directory contains the durable documentation for humans and agents.

## Read first

1. [`CONTEXT.md`](../CONTEXT.md) defines the domain language.
2. [`docs/adr/`](adr/) contains architectural decisions that are hard to reverse.
3. [`docs/architecture/`](architecture/) describes the current structure and flows.
4. [`docs/contracts/`](contracts/) defines schemas, states, and actions.
5. [`docs/operations/`](operations/) documents setup, commands, and recovery.
6. [`docs/research/`](research/) contains cited investigations.
7. [`docs/specs/`](specs/) contains repository-local specifications.

## Documentation rules

- Keep one source of truth for each fact.
- Keep `CONTEXT.md` free of implementation details.
- Add an ADR only for a hard-to-reverse decision with a real trade-off.
- Keep research evidence separate from accepted product and architecture decisions.
- Update this index when a new durable documentation area is added.
- Create subdirectories only when the first document in that category is needed.
