# 0004 — M2 surface capture: stdout-as-framebuffer (the swallow-stage MVP)

**Status**: Accepted
**Date**: 2026-07-03

## Context

M2's acceptance is that a foreign app's rendered surface reaches the compositor.
The consumer side landed first: aethersafha 0.4.0's `src/foreign.cyr` builds a
`MehmanGuestSpec` + an XRGB8888 `MehmanSurface` over a pixel buffer it allocates,
registers a compositor window backed by that buffer, and runs the guest via
`mehman_sandbox_run_guest`. Its own roadmap names the remaining piece: *"capture
the guest framebuffer into the surface buffer (mehman M2 handoff)."* The buffer is
allocated with the explicit comment *"mehman captures the guest into it"* — but
nothing fills it, because the capture is mehman's and did not exist.

The hard constraint: a guest runs under kavach's **PROCESS backend**, which
`fork+exec+waitpid`s the foreign binary and captures its **stdout** — exposed on
`ExecResult` as a **NUL-terminated cstr**, with no separate byte-length accessor.
There is no shared-memory framebuffer channel from a sandboxed guest today.

## Decision

Add `mehman_sandbox_capture_guest(spec, surface, out_exit_code)` (`src/sandbox.cyr`)
and a dependency-free `surface_blit_bytes(surface, src, len)` (`src/surface.cyr`).
Capture runs the guest (sharing `mehman_sandbox_run_guest`'s driver), then blits
the guest's captured **stdout into the surface's pixel buffer** — the swallow-stage
MVP treats **stdout as the guest's framebuffer**. `surface_blit_bytes` copies up to
`len` bytes, clamps to the buffer size (`stride * height`), and zero-fills the
uncovered tail (black). The captured length is `strlen(stdout)`.

This establishes the M2 handoff seam end-to-end: mehman runs a foreign binary
sandboxed and delivers bytes into the compositor's surface buffer.

## Consequences

- **Positive** — M2's data path is real and testable (`/bin/echo AB` → `"AB\n"` in
  the surface buffer). aethersafha can switch `foreign_run` to
  `mehman_sandbox_capture_guest` and its hosted window is backed by live guest
  output. `surface_blit_bytes` is dep-free, so a consumer can also drive it.
- **Negative** — stdout-as-framebuffer is a placeholder for real pixels: binary
  XRGB8888 with embedded NUL bytes **truncates at the first NUL** (`strlen`), and a
  real GUI app (xterm) does not write its framebuffer to stdout at all. This is a
  seam, not fidelity.
- **Neutral** — true framebuffer capture is a future refinement, and needs one of:
  kavach exposing the raw stdout **byte count** (so binary data isn't NUL-clipped),
  or a **shared-memory** handoff backend (the guest renders into a buffer mehman
  maps), or a per-ABI **shim** (M3) that draws the guest's output into the surface.

## Alternatives considered

- **Block M2 until real pixel capture exists.** Rejected — the seam (API + data
  flow + compositor window backing) is the valuable, unblocking part; aethersafha
  is waiting on exactly this shape, and fidelity can improve behind a stable API.
- **Read a fixed `stride * height` bytes instead of `strlen`.** Rejected — kavach
  gives no byte count, and over-reading past the NUL-terminated cstr reads
  unrelated memory. `strlen` is safe; the truncation is documented.
- **Put capture in a new `src/capture.cyr`.** Deferred — it is a few lines sharing
  `sandbox.cyr`'s kavach driver; a separate module isn't warranted yet.
