# 0003 — Feature-gate kavach so mehman's surface/types are consumable standalone

**Status**: Accepted (one factual claim corrected by [0007](0007-make-the-sandbox-off-build-real.md) — see the marked block below)
**Date**: 2026-07-03

## Context

The compositor, aethersafha, is the intended consumer of mehman's surface
contract (it imports mehman's types). But through 0.2.x, aethersafha **deferred**
consuming mehman ("Bite G") for a concrete reason recorded in its own manifest and
changelog: mehman's `[deps.kavach]` drags the full `kavach → sandhi → TLS` cascade
(`sandhi_server_*` / `thread_local_*` / `TLS_BACKEND_LIBSSL`) onto **any** consumer
— *"too large a surface for a types-only, unused dep."* `cyrius build` prepends
every `[deps.*]` module, so even consuming mehman's dependency-free
`src/types.cyr` / `src/surface.cyr` pulled the whole sandbox engine.

The Cyrius toolchain already has the tool to fix this: **dependency-model lever 2**
(v6.3.1) — an `optional = true` key on `[deps.<name>]` and a Cargo-style
`[features]` table, with `cyrius build --features / --no-default-features`. An
inactive optional dep is skipped entirely (no clone, no module copy), and — key
here — *transitive `[features]` tables are not parsed*: the **consumer's** manifest
drives the active feature set. The ecosystem simply hadn't migrated to it yet.

## Decision

Make `[deps.kavach]` **`optional = true`**, gated by a default-on **`sandbox`**
feature (`[features] default = ["sandbox"]`, `sandbox = ["kavach"]`). mehman's own
build and tests keep the sandbox host (default active). A downstream consumer that
declares `[deps.mehman]` for the surface/type modules and does **not** enable
`sandbox` never activates kavach — so the kavach → sandhi → TLS bundle never lands
in the consumer. Verified with a clean-room consumer project (a light stdlib +
`[deps.mehman] modules = ["src/types.cyr", "src/surface.cyr"]`, no `sandbox`): it
resolves to 21 lib files with no kavach and builds + runs against `MehmanSurface`.
(A mehman-side `--no-default-features` build is *not* a faithful guard: mehman's
own `lib/` carries a pinned-snapshot `sakshi` that misses `sakshi_span_*` — a
version skew that only a consumer's own light `lib/` avoids.)

> ⛔ **CORRECTED BY [ADR 0007](0007-make-the-sandbox-off-build-real.md) (2026-08-18) —
> the parenthetical above is wrong, and being wrong is why it did damage.** The
> vendored `sakshi` **defines** `sakshi_span_*` fine; there is no version skew.
> The symbols were undefined because `sakshi` was never in `[deps].stdlib`, so
> `cyrius deps` never prepended its `include` — the sidecar of the *active* kavach
> dep was quietly supplying it, and turning the feature off removed the sidecar.
> A version-skew story is unfixable and excuses the gap; a missing manifest line
> is a one-word fix. That excuse is what let this ADR name an entry point
> (`programs/surface_smoke.cyr`) that did not exist, in a configuration that did
> not compile, for six weeks. The mehman-side build **is** now a faithful guard
> and runs in CI; see ADR 0007.

## Consequences

- **Positive** — aethersafha (or any consumer) can now consume mehman's surface
  contract with a light footprint: a clean-room consumer resolves to **21 lib
  files with no kavach / sigil / sandhi / tls / thread_local / async** and builds
  + runs against `MehmanSurface`. The Bite-G blocker is removed on mehman's side.
  mehman is the ecosystem's first feature-deps adopter.
- **Negative** — mehman's *own* default build still carries kavach + the heavy
  stdlib union (the sandbox host needs them); only *consumers* get the light path.
  ⚠ Amended by ADR 0007: mehman's own build no longer *declares* that union — six
  leaves of its own, with kavach's set coming from `dist/kavach.deps`. It still
  links the union in the default configuration; it just no longer maintains a
  hand-copy of the list, which is what had drifted.
- **Neutral** — CI now runs `cyrius lib sync --full` before `cyrius deps` (the
  gitignored `lib/` must be re-vendored, and the newer toolchain no longer
  auto-vendors stdlib on bare `cyrius deps`). Consumers declare their own light
  stdlib; mehman does not force one on them.
  ⛔ **Reversed at v1.0.2** — `cyrius deps` alone vendors the declared set, and
  `--full` was copying the entire 102-module snapshot, hiding exactly the kind of
  undeclared-leaf gap ADR 0007 then found. Removed from both workflows.

## Alternatives considered

- **`requires = [...]` on the optional kavach dep** to also feature-gate its heavy
  stdlib. Rejected for now — the 6.3.38 stdlib registry rejects several leaf names
  mehman declared (`tls`/`async`/… "not in the cyrius stdlib"), and it is
  unnecessary: stdlib is consumer-declared, so the optional dep alone removes the
  cascade for consumers. Mehman's own build keeps its stdlib union.
- **Split a separate `mehman-surface` package** with no kavach dep. Rejected —
  heavier (a new package/repo) than a feature flag on one manifest.
- **Leave it deferred until aethersafha hosts guests.** Rejected — the decoupling
  is cheap, unblocks aethersafha adopting the contract now, and is the documented
  blocker's actual fix.
