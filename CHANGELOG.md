# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

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
