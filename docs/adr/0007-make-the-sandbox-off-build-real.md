# 0007 — Make the sandbox-off build real rather than claimed

**Status**: Accepted
**Date**: 2026-08-18

## Context

[ADR 0003](0003-feature-gate-kavach-for-standalone-surface-consumption.md) put
`[deps.kavach]` behind a default-on `sandbox` feature so a consumer of mehman's
surface/type contract would not inherit the `kavach → sandhi → TLS` cascade. That
decision is sound and load-bearing (see *Decision*, below, for the check). What it
also did — in its own manifest comment and in its ADR text — was **claim a
configuration nobody had built**:

> `cyrius build --no-default-features` (see `programs/surface_smoke.cyr`)

`programs/surface_smoke.cyr` did not exist. Nothing in CI built the configuration.
And the configuration **did not compile** — verified broken at both cyrius
6.5.5/kavach 3.11.0 and 6.5.27/kavach 3.11.14, so this was never a regression; it
had not worked since the feature landed. Two independent faults, either of which
alone is fatal:

1. **`sakshi` was never declared in `[deps].stdlib`.** `lib/sandhi.cyr` calls
   `sakshi_span_enter` / `sakshi_span_exit` / `sakshi_span_depth`. In the default
   build `sakshi` arrives anyway, because the active kavach dep's
   `dist/kavach.deps` sidecar names it. Turn the feature off and there is no
   sidecar — and mehman's own list never had it. Result:
   `refusing to emit binary with 2 reachable undefined function(s)`.
2. **`src/main.cyr` included `lib/kavach.cyr` unconditionally.** With the dep
   inactive that file is never vendored, so the include cannot resolve even after
   fixing (1): `error: cannot open include file: lib/kavach.cyr`.

ADR 0003 half-saw fault (1) and mis-attributed it, recording that a mehman-side
`--no-default-features` build "is *not* a faithful guard: mehman's own `lib/`
carries a pinned-snapshot `sakshi` that misses `sakshi_span_*`". That reading is
wrong in a way worth naming: the vendored `sakshi` **defines** `sakshi_span_*`
perfectly well. The symbols were undefined because the module was never
*declared*, so `cyrius deps` never prepended its `include`. A version-skew story
is unfixable and excuses the gap; a missing manifest line is a one-word fix. The
misdiagnosis is what let the claim stand unbuilt for six weeks.

The deeper cause is the same one that took `cyrius lib sync --full` out of CI at
v1.0.2: **mehman's `[deps].stdlib` was a hand-copied snapshot of kavach's stdlib
list** (31 leaves — `sandhi`, `tls`, `net`, `keccak`, `async`, `thread_local`, …)
maintained beside the real one. `cyrius deps` already reads kavach's sidecar and
pulls those leaves automatically, so the copy bought nothing and was free to
drift. It had drifted.

## Decision

**Support the configuration for real** — declare it, build it, run it, and gate it
in CI — rather than deleting the feature machinery.

Dropping the machinery was a genuine option, and the deciding check was whether
`optional = true` is load-bearing for consumers. It is. `cyrius deps` **does**
walk transitive `[deps.*]` (Phase 3 is a BFS over each resolved dep's own
manifest, 6.5.27 `cbt/deps.cyr`), but it parses `[features]` **once**, from the
consumer's manifest. A consumer with no `[features]` table activates nothing, so
mehman's optional kavach dep is skipped for it entirely. Remove `optional` and
every consumer of mehman's dependency-free types inherits the full cascade again —
exactly the blocker ADR 0003 removed. (aethersafha would not itself notice: it
consumes `src/sandbox.cyr` and declares `[deps.kavach]` on its own. The consumers
who would notice are the ones ADR 0003 was written for.)

Four changes:

1. **`[deps].stdlib` is now mehman's own needs only** — six leaves (`alloc`,
   `assert`, `bench`, `string`, `syscalls`, `vec`), down from 31. kavach's
   transitive set comes from `dist/kavach.deps`, which is generated from kavach
   and cannot drift from it. The declared graph is now honest in *both*
   configurations, and the sandbox-off build needs nothing the default build
   doesn't.
2. **`src/main.cyr` guards the kavach-dependent includes** with
   `#ifndef MEHMAN_NO_SANDBOX` — `lib/kavach.cyr`, `src/sandbox.cyr`,
   `src/guest.cyr`. The light half (`types` / `surface` / `shim`) is
   unconditional.
3. **`programs/surface_smoke.cyr` exists**, sets `#define MEHMAN_NO_SANDBOX 1`,
   and exercises the light half with 25 checks (caps contract, spec validation,
   surface descriptor + blit incl. zero-fill tail, and the full M3 wire). Exit 0
   on success, 1 with a per-failure line on stderr.
4. **CI gates it** as a *separate job* (`surface-only`), which resolves
   `--no-default-features` into its own empty `lib/` and then builds and runs the
   binary.

**The feature has two halves and the toolchain cannot join them.** `[features]`
gates dependency *resolution* only — there is no feature→`#define` bridge; the
table is read solely by the resolver's `_dep_feature_active` gate. So a
sandbox-off build needs `--no-default-features` (manifest half) *and*
`#define MEHMAN_NO_SANDBOX 1` (source half). The define lives in the **entry
file**, not in a CI `-D` flag, so `surface_smoke.cyr` builds to the light chain
however it is invoked; a `-D` would make the wrong build the silent default.

## Consequences

- **Positive** — the configuration ADR 0003 claimed is now executable, and it
  resolves to **18 lib modules** with no kavach / sigil / sandhi / sakshi / tls /
  thread_local / async. The claim is a test rather than a comment.
- **Positive** — `[deps].stdlib` has one owner per leaf. The class of bug that
  produced this (a hand-maintained copy of another project's dep list, silently
  drifting) cannot recur for kavach's leaves, because they are no longer copied.
- **Positive** — the guard already earned its keep on its first run: it reported
  `vec` undefined (`lib/bench.cyr` and `lib/fmt.cyr` call `vec_len`/`vec_get`),
  a second undeclared leaf the default build had been masking with the sidecar.
- **Negative** — the two halves of the switch must be kept in step by hand, and
  nothing enforces it. Building the sandbox-**on** entry point with
  `--no-default-features` still fails, now with the same
  `cannot open include file: lib/kavach.cyr`. This is documented at the include
  chain in `src/main.cyr` with a table of which entry point takes which flags; it
  is not fixable in-tree (it wants a toolchain feature→define bridge).
- **Negative** — CI is one job wider (~2 min).
- **Neutral, and the guard's real boundary** — it catches a kavach reference in
  the light half only on a **reachable** path, where it hard-errors
  (`refusing to emit binary with N reachable undefined function(s)`). An
  *unreachable* one is a warning the build survives, and `--strict` does **not**
  upgrade it (measured, not assumed). That is the right boundary for what this
  guards — dead code referencing kavach is not a broken consumer — but it is not
  total, and `--strict` is deliberately not passed rather than passed for show.
- **Neutral** — running the sandbox-off resolve *locally* leaves 18 modules in a
  `lib/` the default build then under-populates; re-run plain `cyrius deps` after.
  `cyrius.lock` is safe either way (the sandbox-off resolve visits no named dep,
  so the lock write is skipped), but lib/ is vendored additively and never pruned.
  This is why CI runs it as a separate job with its own checkout, and why merging
  the two jobs would make the guard vacuous.

## Alternatives considered

- **Drop the claim** — remove the `sandbox`/`kavach` optional-feature machinery
  and document mehman as requiring kavach. Rejected on the transitive-resolution
  check above: `optional = true` genuinely stops consumers inheriting the
  kavach → sandhi → TLS cascade, and dropping it would re-impose the exact cost
  ADR 0003 was written to remove.
- **Fix only fault (1)** — add `sakshi` to the existing 31-leaf list and stop.
  Rejected: it leaves the sandbox-off build linking kavach's entire stdlib
  cascade, which it does not use, and leaves the hand-copied list free to drift
  again. It also does not fix fault (2), so the build still would not compile.
- **Drive the source half from a CI `-D MEHMAN_NO_SANDBOX=1`** instead of a
  `#define` in the entry file. Rejected: `cyrius build -D` exists and works, but
  it makes the *correct* build conditional on remembering a flag — building
  `surface_smoke.cyr` without it would silently produce the heavy chain and pass.
  The entry file naming its own configuration cannot be forgotten.
- **Split the light half into its own package** (`mehman-surface`). Rejected for
  the same reason ADR 0003 rejected it — heavier than a feature flag on one
  manifest — and it is now strictly less necessary, since the light half is
  built and run on every push.
