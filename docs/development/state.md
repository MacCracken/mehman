# mehman — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

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

- **Cyrius pin**: `6.5.27` (in `cyrius.cyml [package].cyrius`) — matches the active
  wrapper and the ecosystem (kavach 3.11.14 is on the same pin). Build + smoke +
  **109-assert** suite + 3 benches + fuzz green. (Was `6.3.40` through v1.0.0 and
  `6.5.5` through v1.0.1.)
- **kavach**: `3.11.14` (`[deps.kavach].tag`, optional behind the default-on
  `sandbox` feature). Was `3.11.0` — four patches behind kavach 3.11.4's
  exit-code change, which is how the stale `WEXITSTATUS` contract survived a
  release. ⚠ `path = "../kavach"` sits alongside the tag and **wins locally**, so
  the tag is only actually exercised by CI: when bumping, trust CI over a local
  build.
- **`lib/` model**: `cyrius deps` alone, from an empty tree — **69 modules**
  default, **18** with `--no-default-features`. Do not use `cyrius lib sync
  --full`; it copies the whole 102-module snapshot and hides undeclared
  dependencies from the build (removed from CI at v1.0.2). ⚠ Vendoring is
  ADDITIVE — `cyrius deps` never prunes, so a sandbox-off resolve run over a
  default `lib/` still sees all 69 heavy modules. Re-resolve from an empty tree
  when switching configurations; this is why CI runs the two as separate jobs.
- **`[deps].stdlib` is mehman's OWN needs only** — six leaves (`alloc`, `assert`,
  `bench`, `string`, `syscalls`, `vec`). kavach's transitive set arrives from its
  `dist/kavach.deps` sidecar, automatically. ⛔ Do not re-add kavach's leaves by
  hand: that hand-copy is what drifted (missing `sakshi`) and broke the
  sandbox-off build for six weeks (v1.0.2, ADR 0007).
- **Manifest constraint**: `[deps]` must start within the first **4095 bytes** of
  `cyrius.cyml` — `_auto_deps` scans only that prefix, and past it nothing is
  prepended with no diagnostic naming the manifest. Currently at byte **917**.

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
- **Known-benign warnings** from the heavy kavach bundle (documented in kavach's
  README): `duplicate fn` (`syserr_*`/`err_*`, last-def-wins); a
  `lib/sandhi.cyr` arg-count warning on a credential/TLS path M1 does not
  exercise; ~13 MB static scan tables (`CYRIUS_DCE=1` drops the unreachable
  surface).

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
  delivery (persistent /bin/cat guest: encode → deliver → read back). **109 asserts,
  all passing** on `cyrius test`.
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

- **kavach 3.11.14** — sandbox-execution surface (one-shot + persistent-guest),
  **`optional = true`** behind the
  default-on `sandbox` feature (cyrius dependency-model lever 2). Consumed as
  `dist/kavach.cyr` via `[deps.kavach]` (git + `../kavach` path); pulls transitive
  **sigil** through kavach's own `[deps.sigil]`. A downstream consumer that does
  not enable `sandbox` skips it entirely (no kavach → sandhi → TLS cascade).
- stdlib — **six leaves of mehman's own** (`alloc`, `assert`, `bench`, `string`,
  `syscalls`, `vec`); kavach's transitive union arrives from `dist/kavach.deps`.
  This is mehman's build concern only; **consumers declare their own** stdlib.
- `lib/` is gitignored: **`cyrius deps` alone** regenerates it (69 modules) — from
  an empty tree on a pin move. ⛔ Do **not** use `cyrius lib sync --full`: it copies
  the entire 102-module pinned snapshot, so the build resolves modules the manifest
  never declared and a dependency gap that breaks real consumers stays invisible.
  Removed from both CI workflows at v1.0.2. The no-kavach consumer path is
  documented by a clean-room consumer
  (see [ADR 0003](../adr/0003-feature-gate-kavach-for-standalone-surface-consumption.md))
  **and, since v1.0.2, built on every push** by the `surface-only` CI job
  (`cyrius deps --no-default-features` → 18 modules → build + run
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
