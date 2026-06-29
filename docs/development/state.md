# mehman — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.1.0** — scaffolded 2026-06-29 via `cyrius init`. No releases yet.

## Toolchain

- **Cyrius pin**: `6.3.5` (in `cyrius.cyml [package].cyrius`)

## Source

Initial scaffold only.

## Tests

- `tests/mehman.tcyr` — primary suite (smoke + math; passes on `cyrius test`)
- `tests/mehman.bcyr` — benchmark stub (no-op)
- `tests/mehman.fcyr` — fuzz stub

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert

## Consumers

Intended: **aethersafha** (compositor) — sources foreign-app surfaces from
mehman. None wired yet (post-MVP / swallow stage; `bhumi` ships first).

## Next

See [`roadmap.md`](roadmap.md).
