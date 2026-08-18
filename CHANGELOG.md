# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [1.0.2] — 2026-08-18

Toolchain + dependency refresh, one **semantics change inherited from kavach** in a value this repo
documented in seven places as being something else, and the fix for a build configuration this
project has **claimed since 2026-07-03 and never once built**.

### Changed — toolchain pin `6.5.5` → `6.5.27`, kavach `3.11.0` → `3.11.14`

Both to the current ecosystem. Build, smoke, **surface smoke** (25 checks), **109-assert** suite,
3 benches and the fuzz harness all green — `surface_blit_bytes` ~5.85 µs / 4 KiB,
`shim_encode_input` 14 ns, `shim_encode_resize` 14 ns, unchanged from v1.0.0's recorded numbers.
`[deps]` sits at byte 917 of `cyrius.cyml`, inside the 4095-byte window `_auto_deps` scans (see the
note at the end).

### Fixed — `out_exit_code` is the guest's real `WEXITSTATUS`, and had been for four kavach patches

⚠ **The value changed meaning; no mehman source change caused it.** Through v1.0.1
`mehman_sandbox_run_guest` / `mehman_sandbox_capture_guest` documented `out_exit_code` as a *coarse
exec status* — `0` = ran and output captured, `1` = exec failed — and **explicitly not** the guest's
`WEXITSTATUS`. That was accurate when written: kavach's unconfined capture path used the stdlib
`exec_capture`, which waits on the child and discards the status, and `backend_capture_finish`
hardcoded `0`.

**kavach 3.11.4** routed both the confined and unconfined paths through `confine_capture`, which
decodes the wait status. So `/bin/false` now reports **1** and `/bin/ls` on a missing path reports
**2** — the guest's own exit code, which is what a sandbox host should have been reporting all
along. A host that cannot distinguish a failed guest from a successful one is precisely the defect
kavach fixed.

⛔ **It took four patch releases to notice, and the reason is the same one that produced the 1.0.1
incident — inverted.** mehman's declared `[deps.kavach] tag` was **3.11.0**, pinned before the
change, while every local build resolves through `path = "../kavach"`, which wins. So CI compiled
the old kavach and stayed green on the old assertion, while developers compiled a kavach that had
already flipped. At 1.0.1 the path hid *corrected source* behind a *stale tag*; here it hid a
*corrected dependency* behind one. ⭐ Same root cause both times: **a `path` override does not just
pin a version, it disables the tag as a test.**

Updated everywhere the old contract was stated — [`src/sandbox.cyr`](src/sandbox.cyr) (public doc
comment + the store site), [`docs/api.md`](docs/api.md),
[`docs/development/roadmap.md`](docs/development/roadmap.md) (the M1 caveat is now marked resolved),
[`docs/development/state.md`](docs/development/state.md), and
[`tests/mehman.tcyr`](tests/mehman.tcyr), whose assertion now expects `1` and carries a note not to
"restore" the `0`. [ADR 0002](docs/adr/0002-consume-kavach-3.6.0-and-land-m1.md) gets a
**superseding note** rather than a rewrite: its constraint 2 is resolved upstream, but the decision
it recorded on 2026-07-03 stands.

**Consumer impact — low but real.** aethersafha stores the value (`FGN_EXIT`) and exposes it via
`foreign_exit_status()`, but branches only on the `MehmanError` return, so no control flow changes
there. Any consumer branching on `out_exit_code == 0` to mean "the guest ran" must switch to the
`MehmanError` return for that question.

### Changed — CI builds the dependency graph a consumer actually gets

⛔ **`cyrius lib sync --full` ran before `cyrius deps` in both workflows, and it was hiding
dependency gaps.** It was justified as "required since the toolchain stopped auto-vendoring stdlib
on bare `cyrius deps`" — not true at 6.5.27, and keeping it was worse than redundant: `--full`
copies the **entire** pinned snapshot (**102** modules) while `cyrius deps` resolves the **69** the
manifest actually reaches. CI therefore compiled against modules the manifest never asked for, so it
could stay green on a gap that breaks every consumer resolving mehman normally. Removed from
`ci.yml` and `release.yml`; `cyrius deps` alone now vendors 69 modules and everything passes. The
surviving step carries a comment saying not to reinstate it to fix a missing symbol — declare the
module instead.

`lib/` was likewise re-vendored locally from empty, dropping the same 33 leftover modules.

⭐ **It was hiding one, and the next section is it.** The gap this step had been masking was a
missing `sakshi` declaration, which is why the sandbox-off build had never compiled. CI also gained
a second job, `surface-only`, which builds that configuration — see below.

### Fixed — `--no-default-features` did not build, and never had

⛔ **A configuration this project has claimed since 2026-07-03 had never been built once.**
[ADR 0003](docs/adr/0003-feature-gate-kavach-for-standalone-surface-consumption.md) put
`[deps.kavach]` behind a default-on `sandbox` feature so consumers of mehman's surface/type contract
would not inherit the `kavach → sandhi → TLS` cascade, and its manifest comment pointed at
`programs/surface_smoke.cyr` as the sandbox-off entry point. **That file did not exist, no CI job
built the configuration, and the configuration did not compile.** Verified broken at both 6.5.5 /
kavach 3.11.0 and 6.5.27 / kavach 3.11.14 — long-standing, not a regression. Two independent faults,
either alone fatal:

1. **`sakshi` was never declared in `[deps].stdlib`.** `lib/sandhi.cyr` calls `sakshi_span_enter` /
   `sakshi_span_exit` / `sakshi_span_depth`. In the default build the active kavach dep's
   `dist/kavach.deps` sidecar supplies `sakshi` anyway; with the feature off there is no sidecar, and
   mehman's own list never had it → `refusing to emit binary with 2 reachable undefined function(s)`.
2. **`src/main.cyr` included `lib/kavach.cyr` unconditionally.** With the dep inactive that file is
   never vendored → `error: cannot open include file: lib/kavach.cyr`, even after fixing (1).

⭐ **THE ROOT CAUSE IS A HAND-COPIED DEPENDENCY LIST.** `[deps].stdlib` was the dep-free base **plus
a hand-maintained snapshot of kavach's stdlib** (31 leaves — `sandhi`, `tls`, `net`, `keccak`,
`async`, `thread_local`, …). `cyrius deps` already reads kavach's sidecar and pulls those leaves
automatically, so the copy bought nothing and was free to drift from the real list. It had. Same
shape as the `lib sync --full` removal above: **a second source of truth for a dependency graph does
not stay true, and the build that would have caught it was the one nobody ran.**

⚠ **ADR 0003 half-saw fault (1) and mis-attributed it**, recording that a mehman-side
`--no-default-features` build was "not a faithful guard" because mehman's `lib/` carried "a
pinned-snapshot `sakshi` that misses `sakshi_span_*`". The vendored `sakshi` **defines** those
symbols fine; there was no version skew. The module was simply never declared, so its `include` was
never prepended. A version-skew story is unfixable and excuses the gap; a missing manifest line is a
one-word fix. That excuse is what let the claim stand unbuilt for six weeks. ADR 0003 now carries a
correction block.

Fixed by, in order of what each unblocks:

- **`[deps].stdlib` is mehman's own needs only** — six leaves (`alloc`, `assert`, `bench`, `string`,
  `syscalls`, `vec`), down from 31. kavach's transitive set comes from `dist/kavach.deps`, which is
  generated from kavach and cannot drift from it.
- **`src/main.cyr` guards the kavach-dependent includes** with `#ifndef MEHMAN_NO_SANDBOX`
  (`lib/kavach.cyr`, `src/sandbox.cyr`, `src/guest.cyr`). The light half — `types` / `surface` /
  `shim`, the contract aethersafha imports — is unconditional.
- **`programs/surface_smoke.cyr` now exists**: sets `#define MEHMAN_NO_SANDBOX 1` and runs 25 checks
  over the light half (capability contract, spec validation, surface descriptor + blit incl. the
  zero-filled tail, and the full M3 wire). Exit 0, or 1 with a per-failure line on stderr.
- **CI gates it** — the new `surface-only` job. Separate on purpose: `cyrius deps` vendors additively
  and never prunes, so resolving sandbox-off *after* the default resolve would link against all 69
  heavy modules still sitting in `lib/` and make the guard vacuous.

⚠ **THE FEATURE HAS TWO HALVES AND THE TOOLCHAIN CANNOT JOIN THEM.** `[features]` gates dependency
*resolution* only — there is no feature→`#define` bridge (the table is read solely by the resolver's
`_dep_feature_active` gate). A sandbox-off build needs `--no-default-features` **and**
`#define MEHMAN_NO_SANDBOX 1`. The define lives in the entry file, not a CI `-D`, so
`surface_smoke.cyr` builds light however it is invoked. Building the sandbox-**on** entry point
(`programs/smoke.cyr`) with `--no-default-features` still fails, by design; the include chain in
`src/main.cyr` carries a table of which entry takes which flags.

**Verified**: sandbox-off resolves from an empty `lib/` to **18 modules** — no kavach, sigil,
sandhi, sakshi, tls, thread_local or async — builds clean and runs green. The guard bit immediately
on a *second* undeclared leaf, `vec` (`lib/bench.cyr` + `lib/fmt.cyr` call `vec_len`/`vec_get`),
which the sidecar had also been masking. Its boundary, measured rather than assumed: a kavach
reference in the light half hard-fails only on a **reachable** path; an unreachable one is a warning
the build survives, and `--strict` does not upgrade it. See
[ADR 0007](docs/adr/0007-make-the-sandbox-off-build-real.md).

### Note — a manifest hazard worth knowing about

`cyrius build` reaches the dep resolver through `_auto_deps`, which reads only the **first 4095
bytes** of `cyrius.cyml` looking for `[deps]` / `[deps.`. Push the marker past that — by adding a
comment above it — and nothing is prepended, with no diagnostic naming the manifest, while
`cyrius deps` (which reads 32767 bytes) still succeeds and fully populates `lib/`. ⚠ mehman **is
now documenting the dep list in place** — the ceiling stopped being hypothetical the moment this
release rewrote `[deps]` with its own commentary. `[deps]` moved from byte **362** to byte **917**;
all commentary sits BELOW the marker for this reason, and `cyrius.cyml` says so at the top of the
section. Filed upstream from kavach as
`docs/development/issues/2026-08-17-auto-deps-4095-byte-manifest-window.md`.

## [1.0.1] — 2026-08-02

Ships a fix that has been sitting in `src/` uncut since kavach 3.8.2, and which every consumer
resolving mehman **by tag** has been failing on.

### Fixed — the 1.0.0 tag called `backend_name`, which kavach renamed away

kavach **3.8.2** renamed `backend_name` → `os_backend_name` to end a collision with ai-hwaccel's
own `backend_name` (a *different* enum space — `backend_name(Backend.OCI)` → `"intel-npu"`). This
repo's `src/sandbox.cyr:88` was updated to follow, and then **never cut**. So the published 1.0.0
tag still calls the old name, and against any kavach ≥ 3.8.2 that symbol does not exist.

⛔ **THE VERSION FILE AGREED WITH THE TAG WHILE THE SOURCE DID NOT, which is why no drift check
caught it.** `VERSION` said 1.0.0, the consumer declared `tag = "1.0.0"`, and a survey comparing the
two reports a match. The content had moved underneath both. **A version number is only evidence
about the tag if every source change bumps it.**

⚠ **`path` masked it in every local build.** Consumers declare `path = "../mehman"` alongside the
tag and **the path wins**, so every developer compiled the corrected working tree while CI — which
has no sibling checkouts and resolves purely from git — compiled the stale tag. It surfaced as
`undefined function 'backend_name'` in the aethersafha compositor's CI, in a repo that had changed
nothing related. ⭐ The general rule: a `path` override does not just pin a version, it **disables
the tag as a test**, so the first honest build of the declared graph is the one that runs in CI.

### Fixed — this repo did not build at all

`cyrius build` failed with `undefined function 'thread_local_alloc'`. The symbol is in cyrius
6.5.5's stdlib and **not** in 6.3.40's, and the vendored `lib/` snapshot was taken at the old pin —
the stale-vendored-lib trap. kavach's `dist/` is generated against a current toolchain, so it
reasonably expects a current stdlib. Fixed by pin **6.3.40 → 6.5.5** plus `cyrius lib sync --full`.

### Changed — `[deps.kavach]` tag 3.7.0 → 3.11.0

⚠ The declared tag was **3.7.0**, which is the last kavach that still defined `backend_name` — so
this repo's own manifest disagreed with its own source, in the same direction and for the same
reason. Resolving mehman's graph from tags alone would have failed even without a consumer.

### Verified

Build green (`programs/smoke.cyr`); **109 tests pass, 0 fail**.

## [1.0.0] — 2026-07-03

**mehman 1.0** — the swallow stage is complete. A foreign-ABI app runs sandboxed
under kavach, its surface is captured for the compositor, and compositor events
are delivered to it **live**. Roadmap milestones M1–M4 are all in, and every v1.0
criterion is met (frozen + documented API, tests, benchmarks, a green downstream
consumer, complete changelog, security-audit pass).

### Added
- **M3 live event delivery** (`src/guest.cyr`): `guest_start` launches a
  **persistent live guest** (kavach 3.7.0), and `guest_send_input` /
  `guest_send_resize` shim-encode a compositor event and stream it to the guest's
  stdin; `guest_read` reads its output; `guest_evict` terminates + reaps the live
  handle. Completes M3 (delivery was deferred in 0.4.0). **Verified end-to-end**:
  an input event delivered to a running `/bin/cat` guest is read back
  byte-for-byte. See [ADR 0006](docs/adr/0006-m3-live-delivery-persistent-guest.md).
  109 asserts, all passing.
- `docs/api.md` — the **frozen public API** reference (every exported symbol,
  by module + consumption tier).
- `docs/audit/2026-07-03-audit.md` — a **security-audit pass** of the swallow-stage
  trust boundary (no high/medium findings; two documented low-severity streaming
  notes).

### Changed
- `[deps.kavach]` `3.6.1` → **`3.7.0`** — the persistent-guest execution model
  (live stdin/stdout) M3 delivery streams events over.

## [0.5.0] — 2026-07-03

Hardening toward v1.0: real benchmarks of the swallow-stage compute hot paths
(satisfying the "benchmarks captured" v1.0 criterion) and the kavach dependency
aligned to 3.6.1. No swallow-stage API change — the M1–M4 surface is stable, and
the downstream consumer (aethersafha) is green.

### Added
- `docs/benchmarks.md` + a real `tests/mehman.bcyr` — benchmarks of the
  swallow-stage compute hot paths (surface capture blit ~666 MiB/s; M3 event
  translation ~13–14 ns/event), replacing the no-op bench stub. Satisfies the
  v1.0 "benchmarks captured" criterion.

### Changed
- `[deps.kavach]` `3.6.0` → `3.6.1` (kavach's toolchain-pin maintenance release;
  the consumable `dist/kavach.cyr` is byte-identical apart from its version
  header, so this is inert — mehman's 96-assert suite is unchanged).

## [0.4.0] — 2026-07-03

Milestone **M3 protocol shim (foundation) + M4 guest lifecycle** — the swallow
stage's compositor↔guest seam. M3's event *translation* lands (a per-ABI wire;
live delivery deferred), and M4's `MehmanGuest` lifecycle (spawn / map / run /
evict) becomes mehman's real public surface, retiring the scaffold sentinel. With
this, roadmap milestones M1–M4 are all in.

### Added
- **M4 — guest lifecycle** (`src/guest.cyr`): `MehmanGuest`, the single backend
  handle the compositor drives — `guest_spawn` (spec + surface + ABI), `guest_map`,
  `guest_run` (launch under a kavach PROCESS sandbox + capture the surface),
  `guest_evict`, over a `CREATED → MAPPED → RUNNING → EVICTED` state machine (+
  `guest_spec` / `guest_surface` / `guest_state` / `guest_exit_code` accessors).
  **Acceptance met**: a guest is launched and evicted under compositor control —
  verified (`/bin/echo` spawned → run+captured → evicted; an evicted guest rejects
  further map/run). 16 new asserts (96 total).
- **M3 foundation — protocol shim** (`src/shim.cyr`): the compositor↔guest event
  vocabulary (`MehmanInputEvent` + `MehmanInputKind`, `MehmanLifecycle`,
  `MehmanAbi`) and the per-foreign-ABI **translation** — `shim_encode_input` /
  `shim_encode_resize` / `shim_encode_lifecycle` encode input (key / pointer),
  resize, and lifecycle events into the swallow-ABI byte wire, plus the
  `shim_input_key` / `shim_input_pointer_button` / `shim_input_pointer_motion`
  constructors. Dependency-free (a consumer translates events without the sandbox
  surface); grounded in aethersafha's HID-usage / window-geometry model. 22 new
  asserts (80 total).
- **Delivery deferred**: a kavach PROCESS-backend guest is one-shot (no live
  stdin), so streaming the encoded events to a running guest awaits a
  persistent-guest execution model. M3's acceptance ("a guest receives input and
  resizes correctly") lands with delivery — see
  [ADR 0005](docs/adr/0005-m3-shim-event-wire-and-deferred-delivery.md).

### Removed
- The `mehman_scaffold_ok` scaffold sentinel — `src/guest.cyr` is now the real
  public surface, so the library header no longer needs the proof-of-compile stub
  (the roadmap retires it at M4).

## [0.3.1] — 2026-07-03

### Changed
- Toolchain pin `6.3.5` → `6.3.40`, aligning mehman with the ecosystem
  (aethersafha and kavach both on 6.3.x). Build + the 58-assert suite verified
  green under 6.3.40; also clears the 6.3.5-snapshot `sakshi_span_*` version skew.

## [0.3.0] — 2026-07-03

Milestone **M2 — foreign-surface capture**. mehman runs a foreign guest under a
kavach sandbox and captures its surface into the compositor's buffer — the swallow
handoff — and its surface/type contract is now consumable standalone (kavach
feature-gated). aethersafha 0.4.0 hosts foreign apps on this seam.

### Added
- **M2 handoff** — `mehman_sandbox_capture_guest(spec, surface, out_exit_code)`
  (`src/sandbox.cyr`): runs a foreign guest under the kavach PROCESS sandbox **and
  captures its rendered surface into `surface`'s pixel buffer**, then reaps — the
  swallow stage's whole act (run a foreign binary sandboxed, capture its surface,
  hand the populated buffer to the compositor). This is the capture aethersafha
  0.4.0's `foreign.cyr` waits on to back its hosted window with guest content.
  Verified end-to-end: `/bin/echo AB` runs sandboxed and `"AB\n"` lands in the
  surface buffer.
- `surface_blit_bytes(surface, src, len)` (`src/surface.cyr`) — the dependency-free
  byte sink (copy + clamp-to-buffer + zero-fill) the capture path fills.
- **Capture model / caveat**: kavach's PROCESS backend gives the guest's stdout as
  a NUL-terminated cstr, so the swallow-stage MVP treats stdout **as** the guest's
  framebuffer (binary XRGB8888 with embedded NUL truncates at the first NUL). True
  framebuffer capture (shared-memory handoff, or kavach exposing the raw byte
  count) is a future refinement — see [ADR 0004](docs/adr/0004-m2-surface-capture-stdout-as-framebuffer.md).

### Changed
- **Feature-gate kavach** — migrated to cyrius dependency-model lever 2 (v6.3.1):
  `[deps.kavach]` is now `optional = true` behind a default-on `sandbox` feature
  (`[features] default = ["sandbox"]`, `sandbox = ["kavach"]`). mehman's own build
  and tests keep the sandbox host; a **consumer** of the surface/type contract
  (e.g. aethersafha) builds without enabling `sandbox` and gets **no kavach →
  sandhi → TLS cascade**. Verified: a clean-room consumer resolves to 21 lib files
  (no kavach/sigil/sandhi/tls/thread_local/async) and builds + runs against
  `MehmanSurface`. Removes aethersafha's documented "Bite G" blocker. See
  [ADR 0003](docs/adr/0003-feature-gate-kavach-for-standalone-surface-consumption.md).
- CI / release: run `cyrius lib sync --full` before `cyrius deps` (the gitignored
  `lib/` must be re-vendored from the pin; newer toolchains no longer auto-vendor
  stdlib on bare `cyrius deps`).

## [0.2.1] — 2026-07-03

M2 **foundation** (the dep-free surface descriptor) plus a build-hygiene change.
The full M2 milestone (buffer capture + aethersafha handoff) remains v0.3.0.

### Added
- `src/surface.cyr` — **M2 foundation**: the `MehmanSurface` foreign-surface
  descriptor (width / height / pixel format / stride / buffer handle) +
  `MehmanPixelFormat` (XRGB8888, value-aligned with bhumi's `BHUMI_FMT_XRGB8888`
  so no format remap on handoff), plus `surface_new` / `surface_is_valid` /
  `mehman_format_bpp` / `surface_size_bytes`. This is the producer-side surface
  contract aethersafha will import. **Descriptor + validation only** — the buffer
  capture (shared-memory handoff) and the aethersafha handoff are deferred (gated
  on aethersafha's "Bite G" and on a guest producing pixels, not stdout). Adds 15
  asserts (43 total).

### Changed
- Stop committing the resolved `lib/` — it is now gitignored and regenerated by
  `cyrius deps` from the toolchain pin + the tracked `cyrius.lock` (matching
  kavach/sigil/patra). Release tarballs (`git archive`) no longer ship `lib/`;
  CI and consumers run `cyrius deps`. This also trims the vendored footprint to
  mehman's actual 58-file dependency closure (was 99, incl. scaffold cruft).

## [0.2.0] — 2026-07-03

Milestone **M1 — sandboxed foreign host**. mehman now runs a foreign-ABI binary
as a guest inside a kavach sandbox and reaps it cleanly — the swallow stage's
core act. Depends on kavach 3.6.0 (now consumable as a dist bundle).

### Added
- `src/sandbox.cyr` — `mehman_sandbox_run_guest(spec, out_exit_code)`: validates
  the guest against the swallow-stage capability contract, then drives kavach's
  PROCESS-backend lifecycle (`kavach_init` → `config_new` →
  `config_backend(PROCESS)` → `sandbox_create` → `sandbox_transition(RUNNING)` →
  `sandbox_exec` → `sandbox_destroy`), mapping kavach failure signals onto the
  `MehmanError` namespace. **M1 acceptance met**: a trivial foreign binary
  (`/bin/true`, `/bin/false`) runs sandboxed and is reaped cleanly.
- `tests/mehman.tcyr` — M1 host group (spec rejection pre-init; real
  fork+exec+reap of `/bin/true`/`/bin/false`). **28 asserts total, all passing.**

### Changed
- Consume **kavach 3.6.0** via `[deps.kavach]` (git + `../kavach` path,
  `modules = ["dist/kavach.cyr"]`), source-included in `src/main.cyr`'s chain
  after `src/types.cyr`. `[deps].stdlib` expanded to kavach's full transitive
  set (required to link the bundle). See [ADR 0002](docs/adr/0002-consume-kavach-3.6.0-and-land-m1.md).

### Notes
- kavach's PROCESS backend reports a coarse exec status via `out_exit_code`
  (0 = ran + output captured, 1 = exec failed) — **not** the guest's own
  `WEXITSTATUS`. Documented in `src/sandbox.cyr`; true guest-exit propagation is
  a future kavach concern. M2 (foreign-surface capture) remains gated on
  aethersafha.
- Known-benign integration warnings from the heavy kavach bundle: `duplicate fn`
  (`syserr_*`/`err_*`, last-def-wins), a `lib/sandhi.cyr` arg-count warning on a
  credential path M1 does not exercise, and ~13 MB static scan tables.

## [0.1.0] — 2026-07-02

First release. Scaffold + the dependency-free foundation vocabulary the
swallow-stage modules build on. The M1 sandboxed foreign host is designed but
deferred to v0.2.0 (see Notes).

### Added
- Initial project scaffold (`cyrius init --lib --agent`, pin 6.3.5).
- Project identity: sovereign **compat / "swallow"-stage surface backend** for
  the AGNOS compositor (aethersafha) — hosts foreign-app surfaces as guests in a
  kavach sandbox. The legitimate successor to XWayland's *actual* job, done the
  sovereign way (run a foreign-ABI binary sandboxed, capture its surface) —
  **not** an X.Org port, **not** X11→Wayland translation. Post-MVP; `bhumi`
  (the platform backend) ships first.
- Architecture map in `src/main.cyr`: planned `sandbox` (kavach host) /
  `surface` (capture) / `shim` (per-ABI translation) / `guest` (lifecycle)
  modules.
- `src/types.cyr` — foundation vocabulary shared by every swallow-stage module,
  replacing "scaffold only" with a real (dependency-free) surface:
  - `MehmanError` enum + `mehman_err_name` / `mehman_err_print` — a single error
    namespace for all of mehman; diagnostics to stderr via the stdlib writer.
  - `MehmanCaps` bounded capability set + `caps_swallow_default` /
    `caps_is_swallow_valid` — models the swallow-stage contract (no native-
    protocol access; mediated I/O) as a checkable record, not a comment.
  - `MehmanGuestSpec` guest descriptor + `guest_spec_new` / `guest_spec_is_valid`
    — the minimal, validated input the M1 sandbox host will consume.
- `src/main.cyr` now source-includes `src/types.cyr` at the head of the module
  chain (ordered for single-pass forward-reference resolution). The
  `mehman_scaffold_ok` sentinel is retained (the roadmap removes it at M4).
- `tests/mehman.tcyr` — validation suite for the error namespace, capability
  contract (including rejection paths), and guest-spec validation (22 asserts
  total, up from 2).

### Notes
- M1 (`src/sandbox.cyr`, the kavach-sandboxed foreign host) is designed against
  kavach's `sandbox_create` / `sandbox_exec` / `sandbox_destroy` surface but is
  **deferred** to v0.2.0: kavach is not yet consumable as a Cyrius library
  dependency. See [ADR 0001](docs/adr/0001-defer-m1-until-kavach-is-consumable.md)
  and the M1 status note in [`docs/development/state.md`](docs/development/state.md).
