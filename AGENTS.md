# Kamy-Bot

Kamy-Bot is a new, reusable workflow platform. It is separate from Kamy Jemput, which remains a stable production orchestrator.

## Operating rules

- Treat Kamy-Bot as greenfield architecture.
- Import proven contracts and safety lessons from Kamy Jemput selectively. Do not copy its monolithic implementation by default.
- Keep project-specific behavior behind project profiles and adapters.
- Never let two controller versions process the same issue population.
- Keep workflow-run state isolated per project and per controller version.
- Require explicit approval for production mutations, scope changes, and merge policy changes.
- Prefer replay fixtures and real smoke verification over claims that code compiles.
- Read `CONTEXT.md` before changing domain terminology.
- Read relevant files under `docs/adr/`, `docs/architecture/`, `docs/contracts/`, and `docs/operations/` before changing those areas.

## Agent skills

Matt Pocock skills are installed in `.agents/skills/`.

### Issue tracker

GitHub Issues in `mikhaiIy/kamy-bot`. See `docs/agents/issue-tracker.md`.

### Domain docs

Single-context repository with root `CONTEXT.md` and `docs/adr/`. See `docs/agents/domain.md`.

Use the repository-local skills for discovery, domain modeling, architecture, specifications, tickets, and TDD.

Use one top-level workflow router per session. Do not install or invoke duplicate skill names without an explicit comparison plan.

## Documentation

- `CONTEXT.md` is the domain glossary. Do not put implementation details there.
- `docs/README.md` is navigation only.
- `docs/adr/` contains hard-to-reverse decisions with real trade-offs.
- `docs/architecture/` contains current system structure and flows.
- `docs/contracts/` contains schemas and state/action contracts.
- `docs/operations/` contains setup and recovery procedures.
- `docs/research/` contains cited investigations.
- `docs/specs/` contains repository-local specifications when the issue tracker is not sufficient.
