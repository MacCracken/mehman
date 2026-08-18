# mehman — Public API (v1.0)

The frozen public surface of the mehman swallow-stage backend. Every exported
symbol a consumer (aethersafha) depends on is listed here; all are exercised by
`tests/mehman.tcyr`. Two consumption tiers:

- **Dependency-free** (`src/types.cyr`, `src/surface.cyr`, `src/shim.cyr`) — the
  error / capability / guest-spec / surface / event vocabulary. Consumable
  standalone (no kavach); a consumer declares `[deps.mehman] modules = [...]`
  without enabling the `sandbox` feature. ⭐ Since v1.0.2 this tier is **built and
  run on every push** (`programs/surface_smoke.cyr`, the CI `surface-only` job) —
  before that it was a claim, and a broken one; see
  [ADR 0007](adr/0007-make-the-sandbox-off-build-real.md).
- **Sandbox** (`src/sandbox.cyr`, `src/guest.cyr`) — the kavach-backed execution:
  run/capture and the live guest lifecycle. Requires the `sandbox` feature (kavach
  3.7.0).

Pointer-typed values are `i64`. Constructors return `0` on OOM / invalid input.
`MehmanError` codes: `OK`(0) is success; a returned pointer of `0` is failure.

## `src/types.cyr` — foundation vocabulary (dep-free)

| Symbol | Signature | Description |
|---|---|---|
| `MehmanError` | enum | `OK`, `INVALID_SPEC`, `SANDBOX_CREATE_FAILED`, `EXEC_FAILED`, `EXEC_BLOCKED`, `GUEST_REAP_FAILED`, `CAPABILITY_DENIED` |
| `mehman_err_name` | `(code) -> cstr` | human name for a `MehmanError` |
| `mehman_err_print` | `(code, detail) -> 0` | print `mehman: <name>[: <detail>]` to stderr |
| `MehmanCaps` | struct | bounded capability set: `native_protocol`, `mediated_io` |
| `caps_swallow_default` | `() -> MehmanCaps*` | the swallow contract (no native protocol; mediated I/O) |
| `caps_is_swallow_valid` | `(caps) -> i64` | 1 iff native-protocol denied AND I/O mediated |
| `MehmanGuestSpec` | struct | `command` (cstr), `caps` (MehmanCaps*) |
| `guest_spec_new` | `(command, caps) -> MehmanGuestSpec*` | build a guest spec |
| `guest_spec_is_valid` | `(spec) -> i64` | 1 iff non-empty command + valid swallow caps |

## `src/surface.cyr` — foreign-surface descriptor (dep-free)

| Symbol | Signature | Description |
|---|---|---|
| `MehmanPixelFormat` | enum | `XRGB8888` (= bhumi `BHUMI_FMT_XRGB8888`) |
| `MehmanSurface` | struct | `width`, `height`, `format`, `stride`, `buffer` |
| `surface_new` | `(w, h, format, stride, buffer) -> MehmanSurface*` | build a surface descriptor |
| `surface_is_valid` | `(surface) -> i64` | 1 iff known format, sane dims, stride ≥ w·bpp, non-null buffer |
| `mehman_format_bpp` | `(format) -> i64` | bytes per pixel (4 for XRGB8888; 0 unknown) |
| `surface_size_bytes` | `(surface) -> i64` | `stride * height` |
| `surface_blit_bytes` | `(surface, src, len) -> i64` | copy ≤`len` into the buffer (clamp + zero-fill); the capture sink |

## `src/shim.cyr` — M3 event translation (dep-free)

| Symbol | Signature | Description |
|---|---|---|
| `MehmanInputKind` | enum | `INPUT_KEY`, `INPUT_POINTER_BUTTON`, `INPUT_POINTER_MOTION` |
| `MehmanLifecycle` | enum | `LIFECYCLE_MAP/UNMAP/FOCUS/UNFOCUS/CLOSE` |
| `MehmanAbi` | enum | `ABI_SWALLOW` (the per-foreign-ABI extension point) |
| `MehmanInputEvent` | struct | `kind`, `code`, `pressed`, `x`, `y` |
| `shim_input_key` | `(hid_usage, pressed) -> MehmanInputEvent*` | build a key event |
| `shim_input_pointer_button` | `(button, pressed) -> MehmanInputEvent*` | build a pointer-button event |
| `shim_input_pointer_motion` | `(x, y) -> MehmanInputEvent*` | build a motion event |
| `shim_encode_input` | `(abi, ev, out) -> i64` | encode an input event → wire bytes; bytes written (0 = null/unsupported) |
| `shim_encode_resize` | `(abi, w, h, out) -> i64` | encode a resize event → wire |
| `shim_encode_lifecycle` | `(abi, event, out) -> i64` | encode a lifecycle event → wire |

Swallow wire: key `[0x01,code,pressed]`, pointer-button `[0x02,btn,pressed]`,
motion `[0x03,x:u16le,y:u16le]`, resize `[0x10,w:u16le,h:u16le]`, lifecycle
`[0x20,kind]`.

## `src/sandbox.cyr` — kavach-backed execution (`sandbox` feature)

| Symbol | Signature | Description |
|---|---|---|
| `mehman_sandbox_run_guest` | `(spec, out_exit_code) -> MehmanError` | run a guest one-shot under kavach PROCESS backend; reap |
| `mehman_sandbox_capture_guest` | `(spec, surface, out_exit_code) -> MehmanError` | one-shot run + capture the guest's output into the surface buffer (M2) |

`out_exit_code` is the guest's own exit status — `/bin/false` gives 1 — since
mehman v1.0.2 / kavach 3.11.4. ⚠ Through v1.0.1 it was a coarse exec status
(0 = ran + captured, 1 = exec failed) and explicitly **not** `WEXITSTATUS`;
kavach 3.11.4 began decoding the real status on both its capture paths. Use the
`MehmanError` return to decide whether the run succeeded, and this value to
decide what the guest itself reported. See ADR 0002's superseding note.

## `src/guest.cyr` — guest lifecycle + M3 delivery (`sandbox` feature)

| Symbol | Signature | Description |
|---|---|---|
| `MehmanGuestState` | enum | `GUEST_CREATED/MAPPED/RUNNING/EVICTED` |
| `MehmanGuest` | struct | `spec`, `surface`, `abi`, `state`, `exit_code`, `handle` |
| `guest_spawn` | `(command, w, h) -> MehmanGuest*` | build spec + XRGB8888 surface + swallow ABI (CREATED) |
| `guest_map` | `(g) -> MehmanError` | CREATED/MAPPED → MAPPED |
| `guest_run` | `(g) -> MehmanError` | one-shot run + capture (M2); → RUNNING |
| `guest_start` | `(g) -> MehmanError` | **launch a persistent live guest** (M3); → RUNNING, sets `handle` |
| `guest_send_input` | `(g, event) -> MehmanError` | **shim-encode + deliver an input event** to the live guest |
| `guest_send_resize` | `(g, w, h) -> MehmanError` | shim-encode + deliver a resize event |
| `guest_read` | `(g, buf, len) -> i64` | read the live guest's output (bytes, or -1) |
| `guest_evict` | `(g) -> MehmanError` | terminate + reap the live handle; → EVICTED |
| `guest_spec/surface/abi/state/exit_code/handle` | `(g) -> i64` | field accessors |

## Consuming mehman

See the [README](../README.md) "Consume mehman's surface contract" section. The
dep-free tier needs no kavach; the sandbox tier requires the `sandbox` feature and
kavach 3.7.0. Stability: this surface is **frozen at v1.0** — additive changes only
within 1.x.

⚠ **`mehman_sandbox_run_guest` / `mehman_sandbox_capture_guest`: `out_exit_code`
changed meaning at v1.0.2.** It is now the guest's own `WEXITSTATUS` (kavach
3.11.4), not the coarse "ran / didn't" exec status documented through v1.0.1.
Branch on the `MehmanError` return to learn whether the run succeeded; read
`out_exit_code` as the guest's status. Not an API-shape change — a semantics
change inherited from kavach — but callers that tested `out_exit_code == 0` for
success must move that test to the return value.
