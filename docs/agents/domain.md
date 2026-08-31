# Domain documentation

## Before exploring

Read:

- `CONTEXT.md` at the repository root.
- Relevant files under `docs/adr/`.
- Relevant files under `docs/architecture/`, `docs/contracts/`, and `docs/operations/`.

## File structure

Kamy-Bot is a single-context repository:

```text
/
├── CONTEXT.md
├── docs/
│   ├── agents/
│   ├── adr/
│   ├── architecture/
│   ├── contracts/
│   ├── operations/
│   ├── research/
│   └── specs/
└── src/
```

Create documentation files only when a real term, decision, contract, or procedure needs to be recorded.

## Vocabulary

Use the terms defined in `CONTEXT.md`. If a needed term is missing or overloaded, resolve it through domain modeling before using it in code, issue text, or tests.
