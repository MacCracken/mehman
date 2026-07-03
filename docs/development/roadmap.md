# mehman — Roadmap

> Milestone plan through v1.0. State lives in [`state.md`](state.md);
> this file is the sequencing — what ships, in what order, against
> what dependency gates.

## v1.0 criteria

_Define before tagging v0.1.0:_

- [ ] Public API frozen — every exported symbol documented and tested
- [ ] Test coverage adequate for the surface area
- [ ] Benchmarks captured in `docs/benchmarks.md`
- [ ] At least one downstream consumer green
- [ ] CHANGELOG complete from v0.1.0 onward
- [ ] Security audit pass (`docs/audit/YYYY-MM-DD-audit.md`)

## Milestones

### M0 — Scaffold + foundation (v0.1.0) — ✅ shipped 2026-07-02

- `cyrius init` scaffold landed (2026-06-29)
- Doc-tree per [first-party-documentation.md](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/first-party-documentation.md)
- ADRs / architecture notes / guides / examples folders ready
- **Foundation vocabulary** `src/types.cyr` (`MehmanError` / `MehmanCaps` /
  `MehmanGuestSpec` + validators) — the dependency-free surface M1 builds on;
  22-assert validation suite. First release cut 2026-07-02.

> **Gating note**: mehman is **post-MVP** (the swallow stage). Real milestone
> work starts only after [`bhumi`](https://github.com/MacCracken/bhumi) (the
> platform backend) is consumable by aethersafha and the swallow stage is on the
> AGNOS maturity arc. Until then this repo is a scaffolded seam.

### M1 — Sandboxed foreign host (v0.2.0) — ✅ shipped 2026-07-03

`src/sandbox.cyr` — spawn a foreign-ABI binary under a **kavach** sandbox with a
bounded capability set (no native protocol access; mediated I/O only).
Acceptance: a trivial foreign binary runs sandboxed and is reaped cleanly — ✅
met (`mehman_sandbox_run_guest` runs + reaps `/bin/true` / `/bin/false` via
kavach's PROCESS backend; 28-assert suite green).

> **Shipped v0.2.0 (2026-07-03)**: unblocked once kavach **3.6.0** shipped a
> consumable `dist/kavach.cyr` bundle. mehman consumes it via `[deps.kavach]`
> (plus kavach's transitive stdlib) and drives the sandbox lifecycle. The former
> blocker (kavach not consumable) is resolved — see
> [ADR 0002](../adr/0002-consume-kavach-3.6.0-and-land-m1.md), following on from
> [ADR 0001](../adr/0001-defer-m1-until-kavach-is-consumable.md). Caveat: kavach's
> PROCESS backend does not propagate the guest's `WEXITSTATUS`; M1 surfaces a
> coarse exec status only.

### M2 — Foreign-surface capture (v0.3.0)

`src/surface.cyr` — import the guest's rendered buffer (shared-memory handoff)
and expose it to the compositor as a native surface via aethersafha.
Acceptance: a foreign app's framebuffer appears as a compositor surface.
**Dep gate**: aethersafha surface-import API; bhumi presenting.

> **Foundation shipped in v0.2.1 (2026-07-03)**: `src/surface.cyr` defines the
> producer-side `MehmanSurface` descriptor (`MehmanPixelFormat.XRGB8888`,
> width/height/stride/buffer + validators), value-aligned with bhumi's pixel
> model — the contract aethersafha imports. The two gated halves remain deferred
> to the full v0.3.0 milestone:
> the **buffer capture** (a PROCESS-backend guest emits stdout, not pixels — no
> foreign-render path yet) and the **aethersafha handoff** (aethersafha 0.1.0
> defers consuming mehman — "Bite G", post-MVP — with no surface-import API yet).

### M3 — Protocol shim (v0.4.0)

`src/shim.cyr` — per-foreign-ABI translation of input/resize/lifecycle events
between the compositor and the guest. Acceptance: a guest receives input and
resizes correctly.

### M4 — Guest lifecycle (v0.5.0)

`src/guest.cyr` — spawn / map / evict orchestration folded into the single
backend handle aethersafha drives. Acceptance: guests can be launched and
evicted under compositor control (first downstream consumer green). Removes the
`mehman_scaffold_ok` sentinel.

## Out of scope (for v1.0)

- **An X.Org / XWayland port.** mehman hosts foreign-*ABI* apps; it does not implement the X11 wire protocol. AGNOS has no native X11 client corpus.
- **The platform backend.** Pixels-out / events-in / seat against the kernel is [`bhumi`](https://github.com/MacCracken/bhumi)'s job.
- **The native Wayland protocol / client lifecycle.** That stays in aethersafha; mehman only sources foreign surfaces.
- **Un-sandboxed foreign execution.** Foreign apps are *always* kavach-sandboxed — that boundary never collapses.
