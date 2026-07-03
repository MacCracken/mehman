# Architecture Decision Records

Decisions about mehman — what we chose, the context, and the consequences we accept. Use these when a future reader would reasonably ask *"why did we do it this way?"*

## Conventions

- **Filename**: `NNNN-kebab-case-title.md`, zero-padded to four digits. Never renumber.
- **One decision per ADR.** If a decision supersedes a prior one, add a new ADR and set the old one's status to `Superseded by NNNN`.
- **Status lifecycle**: `Proposed` → `Accepted` → (optionally) `Superseded` or `Deprecated`.
- Use [`template.md`](template.md) as the starting point.

## ADR vs. architecture note vs. guide

| Kind | Lives in | Answers |
|---|---|---|
| ADR | `docs/adr/` | *Why did we choose X over Y?* |
| Architecture note | `docs/architecture/` | *What non-obvious constraint is true about the code?* |
| Guide | `docs/guides/` | *How do I do X?* |

## Index

- [0001 — Land the foundation vocabulary dependency-free; defer M1 until kavach is consumable](0001-defer-m1-until-kavach-is-consumable.md) — *Accepted, 2026-07-02*
- [0002 — Consume kavach 3.6.0 as a dist bundle; land the M1 sandboxed foreign host](0002-consume-kavach-3.6.0-and-land-m1.md) — *Accepted, 2026-07-03 (resolves 0001's M1 deferral)*
- [0003 — Feature-gate kavach so mehman's surface/types are consumable standalone](0003-feature-gate-kavach-for-standalone-surface-consumption.md) — *Accepted, 2026-07-03*
