# mehman

> مهمان — *guest*. The foreign app is a guest; mehman is the guest-house.

**Sovereign compat / "swallow"-stage surface backend for the AGNOS compositor** ([aethersafha](https://github.com/MacCracken/aethersafha)), written in [Cyrius](https://github.com/MacCracken/cyrius) with zero external dependencies.

## What it is

mehman is the legitimate successor to XWayland's *actual* job: hosting the surfaces of **foreign apps** — ones that don't speak the AGNOS-native compositor protocol — as guests inside a **kavach** sandbox. It runs a foreign-ABI binary under kavach, captures its surface, and hands it to the compositor as a well-behaved native surface.

This is the **swallow stage** of the AGNOS maturity arc: a compat sandbox for foreign apps, walled off from the native path.

| concern | mehman module | backed by |
|---|---|---|
| sandboxed foreign host | `src/sandbox.cyr` | [kavach](https://github.com/MacCracken/kavach) (sandbox execution) |
| foreign-surface capture | `src/surface.cyr` | shared-buffer import → aethersafha |
| protocol shim | `src/shim.cyr` | per-foreign-ABI translation |
| guest lifecycle | `src/guest.cyr` | spawn / map / evict |

## What it is NOT

- **Not an X.Org port** and **not** "translate X11 → Wayland." AGNOS has no native X11 clients; mehman is about hosting *foreign-ABI* apps, not bridging the X11 wire protocol.
- **Not the platform backend.** Pixels-out / events-in / seat against the kernel is [`bhumi`](https://github.com/MacCracken/bhumi)'s job. mehman is a *client* (surface-source) backend; bhumi is the *platform* the compositor stands on.

mehman is **post-MVP** (the swallow stage). It is scaffolded now so the compositor seam exists; [`bhumi`](https://github.com/MacCracken/bhumi) is built first.

## Build

`lib/` is gitignored and regenerated from the pinned toolchain:

```sh
cyrius lib sync --full                               # vendor the pinned stdlib snapshot
cyrius deps                                          # resolve [deps.*] (kavach, via the sandbox feature)
cyrius build programs/smoke.cyr build/mehman-smoke   # compile-link the smoke
cyrius test                                          # run tests/*.tcyr
```

## Consume mehman's surface contract (no kavach)

The sandbox host (`src/sandbox.cyr` → kavach) sits behind a default-on `sandbox`
feature, so a compositor can consume mehman's dependency-free surface/type contract
**without** pulling the kavach → sandhi → TLS cascade. In the consumer's
`cyrius.cyml`:

```toml
[deps]
stdlib = ["string", "fmt", "alloc", "io", "vec", "str", "syscalls"]  # your own light set

[deps.mehman]
git     = "https://github.com/MacCracken/mehman.git"
path    = "../mehman"
modules = ["src/types.cyr", "src/surface.cyr"]   # surface contract only — no sandbox
```

Do **not** enable mehman's `sandbox` feature (transitive `[features]` are not
parsed, so it stays off by default). `cyrius deps` then materializes
`lib/mehman_types.cyr` + `lib/mehman_surface.cyr` and skips kavach entirely. Use
`MehmanSurface` / `MehmanPixelFormat` + the validators; the pixel format is
value-aligned with bhumi's `BHUMI_FMT_XRGB8888`, so no remap on handoff.

## License

GPL-3.0-only
