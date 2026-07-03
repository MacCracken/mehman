# 0001 — Land the foundation vocabulary dependency-free; defer M1 until kavach is consumable

**Status**: Accepted
**Date**: 2026-07-02

## Context

The first work past the M0 scaffold was scoped as "foundation, then start M1":
(1) replace the `mehman_scaffold_ok` sentinel with a real, dependency-free type
vocabulary, then (2) wire `src/sandbox.cyr` (the M1 sandboxed foreign host) to
kavach so a trivial foreign binary runs sandboxed and is reaped cleanly.

Part (1) is unconstrained — it is pure in-process data modeling. Part (2) depends
on consuming kavach as a Cyrius library. Investigation of the actual toolchain and
the kavach repo established that **kavach is not currently consumable as a
dependency**:

- The ecosystem consumes siblings as a **single-file `dist/<name>.cyr` bundle**
  declared in `[deps.<name>] modules = ["dist/<name>.cyr"]`; `cyrius deps`
  materializes each listed module into the consumer's `lib/` and the consumer
  source-includes it. kavach has **no `dist/kavach.cyr`** and **no `[lib]`
  section**.
- This toolchain (cyrius 6.3.35) has **no bundle generator**: `cyrius distlib`
  does not exist; `cyrius lib` only vendors stdlib; `cyrius package` produces a
  `.ark`. So the bundle cannot be generated in-tree today.
- kavach's `src/main.cyr` is a **program entry point** (its own `main()`, a demo,
  and a top-level `syscall(SYS_EXIT)`) that pulls 40+ siblings via **relative
  includes**. Pointing `[deps.kavach] modules = ["src/main.cyr"]` at it
  materializes only that one file; a build then fails with
  `error: cannot open include file: src/util.cyr` (verified by a reverted spike).
  The spike did confirm `cyrius deps` transitively resolves kavach's own sigil
  dependency.

Making kavach consumable (a library aggregation header with no `main`/demo, plus
a committed `dist/kavach.cyr` bundle and a `[lib]` section) is **work in the
kavach repo**, and carries its own integration risk (kavach is heavy: ~13 MB of
static scan tables, a transitive sigil dep, and possible stdlib symbol clashes).
That crosses a project boundary and deserves a deliberate, separately-scoped
effort.

## Decision

Land the foundation vocabulary **dependency-free** in a new `src/types.cyr`
(`MehmanError`, `MehmanCaps`, `MehmanGuestSpec`, plus their constructors and
validators), source-included first in `src/main.cyr`'s module chain. **Defer M1**
(`src/sandbox.cyr`, the kavach-sandboxed host) until kavach is consumable as a
Cyrius library. Keep `VERSION` at `0.1.0` (M1 = v0.2.0 is not achieved) and retain
the `mehman_scaffold_ok` sentinel (the roadmap removes it at M4).

Scope in: the type/validation surface and its tests. Scope out: any `src/sandbox.cyr`
code, any `[deps.kavach]` entry, and any modification to the kavach repo.

## Consequences

- **Positive** — mehman advances from a single sentinel to a real, tested public
  surface (22 asserts, all green) with zero new dependencies and no half-wired,
  uncompilable module in the tree. The M1 design is captured (in
  `docs/development/state.md`) so implementation is mechanical once the gate lifts.
- **Negative** — M1's acceptance ("a trivial foreign binary runs sandboxed and is
  reaped cleanly") is not met; the swallow stage still cannot host a guest. The
  real blocker (kavach consumability) is pushed to a future effort rather than
  solved now.
- **Neutral** — creates a follow-on task in the kavach repo: expose a library
  aggregation header + `dist/kavach.cyr` bundle. Surfacing the toolchain's missing
  bundle-generator ("distlib") as an ecosystem gap is a further, larger follow-on.

## Alternatives considered

- **Wire M1 now via `[deps.kavach] modules = ["src/main.cyr"]`** (the pre-spike
  plan). Rejected — empirically does not compile: kavach's `src/main.cyr` is a
  program with relative includes that the dep mechanism does not vendor.
- **Make kavach consumable now** (add the header + bundle in the kavach repo, then
  finish M1). Deferred, not rejected — it is the path to unblock M1, but it
  modifies another repo's release surface and carries unverified integration risk;
  it should be its own deliberate change, not bundled into mehman's first
  post-scaffold increment.
- **Add a bundle-generator to the cyrius toolchain first.** Deferred — fixes the
  root cause but is a large detour from mehman.
- **Fold the vocabulary into `src/main.cyr`.** Rejected — `main.cyr` is the
  include-chain header; the house pattern keeps domain types in their own module
  (mirrors kavach's `error.cyr`/`lifecycle.cyr`), and a dedicated `src/types.cyr`
  lets the future sandbox/surface/shim/guest modules share it cleanly.
