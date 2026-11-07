# mehman — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**1.1.1** — cut 2026-11-06. Newest toolchain and deps: pin `6.7.9` → `6.7.18`, kavach `3.14.0` →
`3.14.6`, sigil `3.13.11` → `3.13.15`. A one-shot guest is now killed at kavach's 30 s deadline, and
mehman returns the new `MehmanError.GUEST_TIMED_OUT`, not `OK`; see Toolchain and
[ADR 0008](../adr/0008-report-a-deadline-kill-as-its-own-error.md). `lib/` resolves to 77 modules (20
sandbox-off). Build, smoke, surface smoke, **123 asserts**, 3 benches and fuzz all green, run as CI
runs them and on `ubuntu:26.04`.

**1.1.0** — cut 2026-11-06. Toolchain moves to the 6.7 language minor: pin `6.6.20` → `6.7.9`.
kavach stays `3.14.0`, now on the same pin, and sigil is `3.13.11` in the build as well as the
lock. No source change, public API unchanged. `path = "../kavach"` is a dev override now, so a plain
local build is the tag CI builds. Build, smoke, surface smoke, **109 asserts**, 3 benches and fuzz
all green, run as CI runs them and on `ubuntu:26.04`.

**1.0.5** — cut 2026-11-06. Toolchain + dependency refresh: pin `6.6.10` → `6.6.20`, kavach
`3.13.1` → `3.14.0`, sigil `3.12.18` → `3.13.10` in the build. No source change. kavach 3.14.0 is
what lets guests run on Ubuntu 25.10+ (uutils); see Toolchain. `lib/` resolves to 69 modules (20
sandbox-off) and `dist/mehman.deps` is gone. Build, smoke, surface smoke, **109 asserts**, 3 benches
and fuzz all green, run as CI runs them and on `ubuntu:26.04`.

**1.0.4** — cut 2026-09-29. kavach → `3.13.1`, pin `6.6.2` → `6.6.10`: fixes two `struct
SpawnedProcess` layouts, and two stale kavach enum spellings in `src/sandbox.cyr`.

**1.0.3** — cut 2026-09-11. Pin `6.5.27` → `6.6.2`, no source change.

**1.0.2** — cut 2026-08-18. Toolchain + dependency refresh: pin `6.5.5` → `6.5.27`,
kavach `3.11.0` → `3.11.14`. Carries one semantics change inherited from kavach —
`out_exit_code` is now the guest's real `WEXITSTATUS` (see the resolved caveat
below). CI stopped running `cyrius lib sync --full`, so it now builds the same
69-module dependency graph a consumer resolving mehman actually gets, instead of
the full 102-module snapshot that was masking gaps. ⭐ And the **sandbox-off build
now exists**: `--no-default-features` had never compiled since the feature landed
(ADR 0003, 2026-07-03) — `[deps].stdlib` was a hand-copy of kavach's list missing
`sakshi`, and `src/main.cyr` included `lib/kavach.cyr` unconditionally. Both
fixed, `programs/surface_smoke.cyr` added, CI gates it. Build, smoke, surface
smoke, **109 asserts**, 3 benches and fuzz all green. See
[ADR 0007](../adr/0007-make-the-sandbox-off-build-real.md).

**1.0.1** — cut 2026-08-02. Cut the `os_backend_name` fix that had been sitting
uncut in `src/` since kavach 3.8.2, breaking every consumer resolving by tag.

**1.0.0** — cut 2026-07-03. **The swallow stage is complete.** A foreign-ABI app
runs sandboxed under kavach (M1), its surface is captured for the compositor (M2),
compositor events translate + deliver to a **live** guest (M3), and the
`MehmanGuest` lifecycle drives spawn/map/run/start/evict (M4). All six v1.0
criteria met — frozen + documented API ([`../api.md`](../api.md)), tests,
benchmarks, a green downstream consumer (aethersafha), complete CHANGELOG, and a
[security audit](../audit/2026-07-03-audit.md). Depends on **kavach 3.7.0**
(persistent-guest model). Builds on 0.5.0 / 0.4.0 (M3+M4) / 0.3.x (M2, pin) /
0.2.0 (M1) / 0.1.0 (scaffold + foundation).

## Toolchain

- **Cyrius pin**: `6.7.18` (in `cyrius.cyml [package].cyrius`). Build + smoke +
  **123-assert** suite + 3 benches + fuzz green.
- **kavach**: `3.14.6` (`[deps.kavach].tag`, optional behind the default-on
  `sandbox` feature), pinned to cyrius 6.7.18.
- **sigil**: `3.13.15`, declared by kavach and shipped by the 6.7.18 stdlib snapshot,
  so the two agree: `lib/sigil.cyr` is byte-identical to the tag.
- **Guest deadline.** mehman sets no timeout, so a one-shot guest gets kavach's 30 s
  default, and since kavach 3.14.4 that holds even for a guest that closes its
  output. A guest killed there returns `MehmanError.GUEST_TIMED_OUT` with
  `out_exit_code` 137 and its output captured. kavach's gate runs first, so a killed
  guest whose output is quarantined returns `EXEC_BLOCKED`. `guest_start` guests have
  no deadline. ⚠ The gate scans guest stdout, so a test guest must not print host
  files: `/etc/passwd` passes on one host and is quarantined on another.
- **`path = "../kavach"` is a dev override** (cyrius >= 6.7.6). A default build
  resolves the tag, checks its commit pin and prints a `hint:` naming the skipped
  checkout, so a plain local build is what CI builds. `--local` / `CYRIUS_LOCAL=1`
  builds the sibling into `build/local-deps/` without writing `lib/` or
  `cyrius.lock`. A tag bump keeps the old tag's commit row; delete `cyrius.lock` and
  run `cyrius deps` to drop it.
- **uutils guests** (fixed v1.0.5). On kavach 3.12.7 – 3.13.x a pinned exec named its
  program `/dev/fd/N` and uutils' multicall binary refused to run, so a uutils guest
  exited 1 with no output, on both mehman guest paths (`confine_capture`,
  `persistent_spawn`). Ubuntu 26.04, behind `ubuntu-latest`, has `/bin/echo`,
  `/bin/cat` and `/bin/false` as uutils and `/bin/true` as GNU. kavach 3.14.0 execs
  through `/proc/self/fd/N`. To check a uutils host locally, build on the host and
  run the static binary in a container:
  `docker run --rm -v "$PWD/build:/t:ro" ubuntu:26.04 sh -c /t/<binary>`.
- **`lib/` model**: `cyrius deps` alone, from an empty tree (`rm -rf lib`: it now
  holds `lib/unicode/`) — **77 modules** default, **20** with
  `--no-default-features`. Do not use `cyrius lib sync
  --full`; it copies the whole 102-module snapshot and hides undeclared
  dependencies from the build (removed from CI at v1.0.2). ⚠ Vendoring is
  ADDITIVE — `cyrius deps` never prunes, so a sandbox-off resolve run over a
  default `lib/` still sees all 77 heavy modules. Re-resolve from an empty tree
  when switching configurations; this is why CI runs the two as separate jobs.
- **`[deps].stdlib` is mehman's OWN needs only** — six leaves (`alloc`, `assert`,
  `bench`, `string`, `syscalls`, `vec`). kavach's transitive set arrives from its
  `dist/kavach.deps` sidecar, automatically. ⛔ Do not re-add kavach's leaves by
  hand: that hand-copy is what drifted (missing `sakshi`) and broke the
  sandbox-off build for six weeks (v1.0.2, ADR 0007).

## Source

- `src/main.cyr` — library header / module-include chain: `src/types.cyr` →
  `lib/kavach.cyr` (dist bundle) → `src/sandbox.cyr` → `src/surface.cyr` →
  `src/shim.cyr` → `src/guest.cyr`. The `mehman_scaffold_ok` sentinel is **retired**
  (guest.cyr is the real public surface). ⚠ The kavach-dependent three
  (`lib/kavach.cyr`, `sandbox`, `guest`) sit behind `#ifndef MEHMAN_NO_SANDBOX` —
  the **source half** of the `sandbox` feature, which `[features]` cannot set for
  you (it gates dep resolution only; there is no feature→`#define` bridge). A
  sandbox-off build needs `--no-default-features` AND that define; the include
  chain carries a table of which entry point takes which.
- `src/types.cyr` — **foundation vocabulary** (dependency-free):
  - `MehmanError` enum + `mehman_err_name` / `mehman_err_print`.
  - `MehmanCaps` (bounded capability set) + `caps_swallow_default` /
    `caps_is_swallow_valid` — the swallow-stage contract (no native-protocol
    access; mediated I/O) as a validatable record.
  - `MehmanGuestSpec` (guest descriptor) + `guest_spec_new` /
    `guest_spec_is_valid` — the validated input the M1 host consumes.
- `src/sandbox.cyr` — **M1 host + M2 capture**: `mehman_sandbox_run_guest(spec,
  out_exit_code)` drives kavach's PROCESS-backend lifecycle (`kavach_init` →
  `config_new` → `config_backend(PROCESS)` → `sandbox_create` →
  `sandbox_transition(RUNNING)` → `sandbox_exec` → `sandbox_destroy`), mapping
  kavach failures onto `MehmanError`. `mehman_sandbox_capture_guest(spec, surface,
  out_exit_code)` adds the **M2 handoff** — same run, plus blitting the guest's
  output into the surface buffer. Both share a private `_mehman_run_guest` driver.
- `src/surface.cyr` — **M2 surface** (dependency-free): the `MehmanSurface`
  descriptor (width / height / format / stride / buffer) + `MehmanPixelFormat`
  (XRGB8888) + `surface_new` / `surface_is_valid` / `mehman_format_bpp` /
  `surface_size_bytes` + `surface_blit_bytes` (the capture byte sink). The
  producer-side surface contract aethersafha imports, value-aligned with bhumi's
  XRGB8888 pixel model.
- `src/shim.cyr` — **M3 foundation** (dependency-free): the compositor↔guest event
  vocabulary (`MehmanInputEvent` + `MehmanInputKind`, `MehmanLifecycle`,
  `MehmanAbi`) + per-ABI encoders `shim_encode_input` / `shim_encode_resize` /
  `shim_encode_lifecycle` (translate events → the swallow-ABI byte wire) +
  `shim_input_key` / `shim_input_pointer_button` / `shim_input_pointer_motion`.
  Live delivery to a running guest is deferred (one-shot exec) — see M3 status.
- `src/guest.cyr` — **M4 guest lifecycle + M3 delivery**: `MehmanGuest`, the single
  backend handle the compositor drives — `guest_spawn` (spec + surface + ABI) /
  `guest_map` / `guest_run` (one-shot launch + capture) / `guest_evict`, over a
  CREATED → MAPPED → RUNNING → EVICTED state machine (+ accessors). Plus the **M3
  live-delivery** path: `guest_start` (persistent kavach guest), `guest_send_input`
  / `guest_send_resize` (shim-encode + stream to the live guest), `guest_read`. The
  real public surface (retires the sentinel). Rides the `sandbox` feature (kavach).

## M1 status — shipped (v0.2.0)

M1 acceptance is **met**: `mehman_sandbox_run_guest` runs `/bin/true` and
`/bin/false` as guests under kavach's PROCESS backend (real fork+exec+waitpid)
and reaps them cleanly (`MehmanError.OK`). Unblocked once **kavach 3.6.0** shipped
a consumable `dist/kavach.cyr` bundle (the former blocker — kavach not consumable
as a Cyrius library — is resolved). See
[ADR 0002](../adr/0002-consume-kavach-3.6.0-and-land-m1.md).

- **Resolved caveat (v1.0.2)**: kavach's PROCESS backend now propagates the
  guest's own `WEXITSTATUS`. It used to report a coarse exec status (0 = ran +
  captured, 1 = exec failed) because the unconfined path went through the stdlib
  `exec_capture`, which discards the child's status; **kavach 3.11.4** routed
  both the confined and unconfined paths through `confine_capture`, which decodes
  it. mehman's `out_exit_code` is therefore the guest's real exit code.
  ⚠ The stale contract survived a release because mehman's declared
  `[deps.kavach] tag` was **3.11.0**, four patches behind the change, while local
  builds resolved through `path = "../kavach"` — the same tag-vs-path split that
  produced the 1.0.1 incident, in the opposite direction.
- **Known-benign warnings** at this pin: the `sigil` stdlib-leaf note from `cyrius
  deps` and ~663 KB of static data (`CYRIUS_DCE=1` drops the unreachable surface). **0 `duplicate fn`** across the smoke, test, bench
  and fuzz units.

## M2 status — shipped (v0.3.0)

The M2 handoff is implemented: `mehman_sandbox_capture_guest(spec, surface,
out_exit_code)` (`src/sandbox.cyr`) runs a foreign guest under the kavach PROCESS
sandbox and captures its output into the surface's pixel buffer via the dep-free
`surface_blit_bytes`. Verified end-to-end (`/bin/echo AB` → `"AB\n"` in the
buffer). aethersafha **0.4.0** already has the compositor side (`src/foreign.cyr`:
guest spec + XRGB8888 surface + hosted window + `foreign_run`) and consumes
mehman's `types`/`surface`/`sandbox` modules directly (with its own `[deps.kavach]`);
it switches `foreign_run` to `mehman_sandbox_capture_guest` to back its hosted
window with live guest content.

- **Capture model** is **stdout-as-framebuffer** (MVP seam): kavach's PROCESS
  backend gives stdout as a NUL-terminated cstr, so binary XRGB8888 truncates at
  the first NUL. Real pixel fidelity (shared-memory handoff, a kavach byte-count,
  or an M3 per-ABI shim) is the follow-on. See
  [ADR 0004](../adr/0004-m2-surface-capture-stdout-as-framebuffer.md).
- The descriptor + format stay value-aligned with bhumi's XRGB8888 model, so no
  remap on handoff.

## M3 status — complete (v1.0.0)

Both halves are done. **Translation** (`src/shim.cyr`, dep-free): per-ABI encoders
turn compositor input/resize/lifecycle events into the swallow-ABI byte wire.
**Live delivery** (`src/guest.cyr`): `guest_start` launches a persistent live guest
(kavach's persistent-guest model, added 3.7.0), `guest_send_input` /
`guest_send_resize` shim-encode + stream events
to its stdin, `guest_read` reads its output. M3's acceptance ("a guest receives
input and resizes correctly") is **met** — verified end-to-end: an input event
delivered to a running `/bin/cat` guest is read back byte-for-byte. See
[ADR 0005](../adr/0005-m3-shim-event-wire-and-deferred-delivery.md) (translation)
and [ADR 0006](../adr/0006-m3-live-delivery-persistent-guest.md) (delivery).

## M4 status — shipped (v0.4.0)

M4 **acceptance is met**: a guest is launched and evicted under compositor
control. `src/guest.cyr`'s `MehmanGuest` bundles the spec (M1) + surface (M2) +
ABI (M3) behind `guest_spawn` → `guest_map` → `guest_run` (launch under kavach +
capture) → `guest_evict`, a `CREATED → MAPPED → RUNNING → EVICTED` state machine.
Verified: `/bin/echo` spawned, run+captured into its surface, and evicted; an
evicted guest rejects further map/run. This is mehman's real public surface — the
`mehman_scaffold_ok` sentinel is retired. aethersafha can drive a `MehmanGuest`
per hosted foreign app (its `src/foreign.cyr` currently orchestrates the same
pieces directly; adopting the handle is a follow-on).

## Tests

- `tests/mehman.tcyr` — primary suite: smoke + foundation validation (error
  namespace, capability contract incl. rejection paths, guest-spec validation) +
  M1 host (spec rejection pre-init; real run+reap of `/bin/true`/`/bin/false`) +
  M2 surface descriptor + `surface_blit_bytes` + the M2 capture handoff (real
  `/bin/echo` output captured into a surface buffer) + M3 event translation
  (per-ABI input/resize/lifecycle wire, byte-checked) + M4 guest lifecycle
  (spawn/map/run/evict state machine, real launch+capture+evict) + M3 live
  delivery (persistent /bin/cat guest: encode → deliver → read back) + the guest
  deadline (real guests killed at a 200 ms deadline, via the private
  `_mehman_deadline_ms`). **123 asserts, all passing** on `cyrius test`.
- `tests/mehman.bcyr` — **benchmarks** of the compute hot paths (capture blit
  ~666 MiB/s; shim event translation ~13–14 ns/event); results in
  [`../benchmarks.md`](../benchmarks.md). Run: `cyrius bench tests/mehman.bcyr`.
- `tests/mehman.fcyr` — fuzz stub
- `programs/surface_smoke.cyr` — **the sandbox-off guard** (v1.0.2): 25 checks over
  the light half (`types` / `surface` / `shim`) with kavach absent from both the
  manifest and the include chain. Run:
  `cyrius deps --no-default-features && cyrius build --no-default-features
  programs/surface_smoke.cyr build/mehman-surface-smoke` — from an empty `lib/`.
  ⚠ Its reach: a kavach reference in the light half hard-fails only on a
  **reachable** path; an unreachable one is a warning the build survives, and
  `--strict` does not upgrade it (measured).

## Dependencies

Direct (declared in `cyrius.cyml`):

- **kavach 3.14.6** — sandbox-execution surface (one-shot + persistent-guest),
  **`optional = true`** behind the
  default-on `sandbox` feature (cyrius dependency-model lever 2). Consumed as
  `dist/kavach.cyr` via `[deps.kavach]` (git + `../kavach` path); pulls transitive
  **sigil** through kavach's own `[deps.sigil]`. A downstream consumer that does
  not enable `sandbox` skips it entirely (no kavach → sandhi → TLS cascade).
- stdlib — **six leaves of mehman's own** (`alloc`, `assert`, `bench`, `string`,
  `syscalls`, `vec`); kavach's transitive union arrives from `dist/kavach.deps`.
  This is mehman's build concern only; **consumers declare their own** stdlib.
- `lib/` is gitignored: **`cyrius deps` alone** regenerates it (77 modules) — from
  an empty tree on a pin move. ⛔ Do **not** use `cyrius lib sync --full`: it copies
  the entire 102-module pinned snapshot, so the build resolves modules the manifest
  never declared and a dependency gap that breaks real consumers stays invisible.
  Removed from both CI workflows at v1.0.2. The no-kavach consumer path is
  documented by a clean-room consumer
  (see [ADR 0003](../adr/0003-feature-gate-kavach-for-standalone-surface-consumption.md))
  **and, since v1.0.2, built on every push** by the `surface-only` CI job
  (`cyrius deps --no-default-features` → 20 modules → build + run
  `programs/surface_smoke.cyr`). See
  [ADR 0007](../adr/0007-make-the-sandbox-off-build-real.md).

## Consumers

Intended: **aethersafha** (compositor) — sources foreign-app surfaces from mehman.
As of the feature-gate migration (see [ADR 0003](../adr/0003-feature-gate-kavach-for-standalone-surface-consumption.md)),
aethersafha **can** consume mehman's surface/type contract (`src/types.cyr` +
`src/surface.cyr`) with a light footprint and no kavach cascade — verified by a
clean-room consumer. Not yet wired on aethersafha's side (its "Bite G"): it still
lacks a surface-import API, and a guest producing pixels (not stdout) is absent.

## Next

See [`roadmap.md`](roadmap.md). M2 — foreign-surface capture (`src/surface.cyr`),
gated on the aethersafha surface-import API.
