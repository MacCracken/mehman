# mehman — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.3.1** — cut 2026-07-03. Toolchain pin bump to 6.3.40 (ecosystem alignment) on
top of **0.3.0** — Milestone **M2 — foreign-surface capture**: mehman runs a
foreign guest and captures its surface into the compositor's buffer
(`mehman_sandbox_capture_guest`), and its surface/type contract is consumable
standalone (kavach feature-gated). Builds on **0.2.1** (M2 foundation — the
surface descriptor), **0.2.0** (M1 — sandboxed foreign host, kavach 3.6.0), and
**0.1.0** (scaffold + foundation vocabulary). Consumed by aethersafha 0.4.0.

## Toolchain

- **Cyrius pin**: `6.3.40` (in `cyrius.cyml [package].cyrius`) — matches the active
  wrapper and the ecosystem (aethersafha, kavach). Build + 58-assert suite green.

## Source

- `src/main.cyr` — library header / module-include chain: `src/types.cyr` →
  `lib/kavach.cyr` (dist bundle) → `src/sandbox.cyr` → `src/surface.cyr` →
  `src/shim.cyr`. Retains the `mehman_scaffold_ok` sentinel (removed at M4).
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

## M1 status — shipped (v0.2.0)

M1 acceptance is **met**: `mehman_sandbox_run_guest` runs `/bin/true` and
`/bin/false` as guests under kavach's PROCESS backend (real fork+exec+waitpid)
and reaps them cleanly (`MehmanError.OK`). Unblocked once **kavach 3.6.0** shipped
a consumable `dist/kavach.cyr` bundle (the former blocker — kavach not consumable
as a Cyrius library — is resolved). See
[ADR 0002](../adr/0002-consume-kavach-3.6.0-and-land-m1.md).

- **Known caveat**: kavach's PROCESS backend captures guest *output* via
  `exec_capture` and reports a coarse exec status (0 = ran + captured, 1 = exec
  failed) — it does **not** propagate the guest's own `WEXITSTATUS`. mehman's
  `out_exit_code` surfaces that coarse status; true guest-exit propagation is a
  future kavach concern.
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

## M3 status — foundation landed (unreleased, toward v0.4.0)

The M3 **translation** is implemented in `src/shim.cyr` (dependency-free): the
event vocabulary + per-ABI encoders that turn compositor input / resize /
lifecycle events into the swallow-ABI byte wire, verified byte-for-byte. The
`MehmanAbi` enum is the per-foreign-ABI extension point.

- **Live delivery is deferred** (the gated half): a kavach PROCESS-backend guest
  is one-shot (no live stdin), so there is no running guest to stream the encoded
  events to yet. M3's acceptance ("a guest receives input and resizes correctly")
  lands with a persistent-guest execution model (a kavach change). See
  [ADR 0005](../adr/0005-m3-shim-event-wire-and-deferred-delivery.md).

## Tests

- `tests/mehman.tcyr` — primary suite: smoke + foundation validation (error
  namespace, capability contract incl. rejection paths, guest-spec validation) +
  M1 host (spec rejection pre-init; real run+reap of `/bin/true`/`/bin/false`) +
  M2 surface descriptor + `surface_blit_bytes` + the M2 capture handoff (real
  `/bin/echo` output captured into a surface buffer) + M3 event translation
  (per-ABI input/resize/lifecycle wire, byte-checked). **80 asserts, all passing**
  on `cyrius test`.
- `tests/mehman.bcyr` — benchmark stub (no-op)
- `tests/mehman.fcyr` — fuzz stub

## Dependencies

Direct (declared in `cyrius.cyml`):

- **kavach 3.6.0** — sandbox-execution surface, **`optional = true`** behind the
  default-on `sandbox` feature (cyrius dependency-model lever 2). Consumed as
  `dist/kavach.cyr` via `[deps.kavach]` (git + `../kavach` path); pulls transitive
  **sigil** through kavach's own `[deps.sigil]`. A downstream consumer that does
  not enable `sandbox` skips it entirely (no kavach → sandhi → TLS cascade).
- stdlib — mehman's own default-build set (kavach's transitive union). This is
  mehman's build concern only; **consumers declare their own** (light) stdlib.
- `lib/` is gitignored: `cyrius lib sync --full` (pinned snapshot) + `cyrius deps`
  regenerate it. The no-kavach consumer path is verified by a clean-room consumer
  (see [ADR 0003](../adr/0003-feature-gate-kavach-for-standalone-surface-consumption.md)).

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
