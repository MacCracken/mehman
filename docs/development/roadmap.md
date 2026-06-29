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

### M0 — Scaffold (v0.1.0) — ✅ shipped 2026-06-29

- `cyrius init` scaffold landed
- Doc-tree per [first-party-documentation.md](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/first-party-documentation.md)
- ADRs / architecture notes / guides / examples folders ready

> **Gating note**: mehman is **post-MVP** (the swallow stage). Real milestone
> work starts only after [`bhumi`](https://github.com/MacCracken/bhumi) (the
> platform backend) is consumable by aethersafha and the swallow stage is on the
> AGNOS maturity arc. Until then this repo is a scaffolded seam.

### M1 — Sandboxed foreign host (v0.2.0)

`src/sandbox.cyr` — spawn a foreign-ABI binary under a **kavach** sandbox with a
bounded capability set (no native protocol access; mediated I/O only).
Acceptance: a trivial foreign binary runs sandboxed and is reaped cleanly.
**Dep gate**: kavach sandbox-execution surface; an AGNOS foreign-ABI execution
path (the swallow stage existing at all).

### M2 — Foreign-surface capture (v0.3.0)

`src/surface.cyr` — import the guest's rendered buffer (shared-memory handoff)
and expose it to the compositor as a native surface via aethersafha.
Acceptance: a foreign app's framebuffer appears as a compositor surface.
**Dep gate**: aethersafha surface-import API; bhumi presenting.

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
