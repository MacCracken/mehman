# mehman — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.2.0** — cut 2026-07-03. Milestone **M1 — sandboxed foreign host**: mehman
runs a foreign-ABI binary as a guest inside a kavach sandbox and reaps it
cleanly. Builds on **0.1.0** (2026-07-02: scaffold + the dependency-free
foundation vocabulary). M2 (foreign-surface capture) is next and gated on
aethersafha.

## Toolchain

- **Cyrius pin**: `6.3.5` (in `cyrius.cyml [package].cyrius`; wrapper is 6.3.36 —
  benign drift, warned on every build).

## Source

- `src/main.cyr` — library header / module-include chain: `src/types.cyr` →
  `lib/kavach.cyr` (dist bundle) → `src/sandbox.cyr` → `src/surface.cyr`. Retains
  the `mehman_scaffold_ok` sentinel (removed at M4).
- `src/types.cyr` — **foundation vocabulary** (dependency-free):
  - `MehmanError` enum + `mehman_err_name` / `mehman_err_print`.
  - `MehmanCaps` (bounded capability set) + `caps_swallow_default` /
    `caps_is_swallow_valid` — the swallow-stage contract (no native-protocol
    access; mediated I/O) as a validatable record.
  - `MehmanGuestSpec` (guest descriptor) + `guest_spec_new` /
    `guest_spec_is_valid` — the validated input the M1 host consumes.
- `src/sandbox.cyr` — **M1 host**: `mehman_sandbox_run_guest(spec, out_exit_code)`
  validates the spec, then drives kavach's PROCESS-backend lifecycle
  (`kavach_init` → `config_new` → `config_backend(PROCESS)` → `sandbox_create` →
  `sandbox_transition(RUNNING)` → `sandbox_exec` → `sandbox_destroy`), mapping
  kavach failures onto `MehmanError`.
- `src/surface.cyr` — **M2 foundation** (dependency-free): the `MehmanSurface`
  descriptor (width / height / format / stride / buffer) + `MehmanPixelFormat`
  (XRGB8888) + `surface_new` / `surface_is_valid` / `mehman_format_bpp` /
  `surface_size_bytes`. The producer-side surface contract aethersafha imports,
  value-aligned with bhumi's XRGB8888 pixel model.

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

## M2 status — foundation landed, capture/handoff gated

`src/surface.cyr` defines the `MehmanSurface` descriptor (the producer-side
contract aethersafha imports) and its validation, dependency-free and shipped as
unreleased groundwork toward v0.3.0. The two gated halves are **deferred**:

- **Buffer capture** — a kavach PROCESS-backend guest produces stdout, not a
  framebuffer; there is no foreign-guest-renders-a-surface path yet.
- **aethersafha handoff** — aethersafha exists (0.1.0) but its `cyrius.cyml`
  defers consuming mehman ("Bite G", post-MVP) and exposes no surface-import API
  yet.

The descriptor is value-aligned with bhumi's XRGB8888 pixel model, so no format
remap is needed once the handoff lands.

## Tests

- `tests/mehman.tcyr` — primary suite: smoke + foundation validation (error
  namespace, capability contract incl. rejection paths, guest-spec validation) +
  M1 host (spec rejection pre-init; real run+reap of `/bin/true`/`/bin/false`) +
  M2 surface descriptor (format bpp, round-trip, validity + rejection paths).
  **43 asserts, all passing** on `cyrius test`.
- `tests/mehman.bcyr` — benchmark stub (no-op)
- `tests/mehman.fcyr` — fuzz stub

## Dependencies

Direct (declared in `cyrius.cyml`):

- **kavach 3.6.0** — sandbox-execution surface, consumed as `dist/kavach.cyr`
  via `[deps.kavach]` (git + `../kavach` path). Pulls transitive **sigil** (and
  its sandhi/sakshi) through kavach's own `[deps.sigil]`.
- stdlib — expanded to kavach's full transitive set (required to link the
  bundle): alloc, assert, async, bayan, bench, chrono, ct, dynlib, fdlopen, fmt,
  fnptr, freelist, fs, hashmap, hashmap_fast, io, keccak, mmap, net, process,
  random, result, sandhi, slice, str, string, syscalls, tagged, thread,
  thread_local, tls, vec.

## Consumers

Intended: **aethersafha** (compositor) — sources foreign-app surfaces from
mehman. None wired yet (post-MVP / swallow stage; `bhumi` ships first).

## Next

See [`roadmap.md`](roadmap.md). M2 — foreign-surface capture (`src/surface.cyr`),
gated on the aethersafha surface-import API.
