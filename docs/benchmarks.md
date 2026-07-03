# mehman — Benchmarks

The swallow-stage **compute** hot paths on the per-frame / per-event path,
measured with `cyrius bench tests/mehman.bcyr` (the stdlib `bench` harness —
batched tight loops that amortize the ~240 ns `clock_gettime` start/stop
overhead). The kavach-sandboxed run/capture itself (`mehman_sandbox_capture_guest`)
is **syscall-bound** (`fork` + `exec` + `waitpid`), not a micro-benchmark, so it
is deliberately not timed here.

## Results

Environment: x86_64 Linux, cyrius 6.3.40.

| Benchmark | avg | iters | what it is |
|---|---|---|---|
| `surface_blit_bytes` (4 KiB capture) | **5.865 µs** | 200,000 | copy + zero-fill 4 KiB into a surface buffer — the M2 capture sink (~666 MiB/s) |
| `shim_encode_input` (key) | **13 ns** | 2,000,000 | translate a key event → the guest wire (3-byte encode) — the M3 shim |
| `shim_encode_resize` | **14 ns** | 2,000,000 | translate a resize event → the guest wire (5-byte u16-LE encode) |

## Reading these

- **Event translation is effectively free** (~13–14 ns/event): the M3 shim adds
  negligible per-event cost — a compositor can encode thousands of input/resize
  events per frame without concern.
- **Capture blit** is the per-frame cost that matters: ~666 MiB/s for the current
  byte-by-byte copy + zero-fill. A 1080p XRGB8888 frame (~7.9 MiB) would take
  ~12 ms at this rate — acceptable for the stdout-as-framebuffer MVP
  ([ADR 0004](adr/0004-m2-surface-capture-stdout-as-framebuffer.md)), and the
  obvious first optimization (word-sized copy, or `memcpy`-class stdlib) once
  real-pixel capture lands.

## Reproduce

```sh
cyrius lib sync --full && cyrius deps
cyrius bench tests/mehman.bcyr
```
