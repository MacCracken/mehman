# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.2.0] — 2026-07-03

Milestone **M1 — sandboxed foreign host**. mehman now runs a foreign-ABI binary
as a guest inside a kavach sandbox and reaps it cleanly — the swallow stage's
core act. Depends on kavach 3.6.0 (now consumable as a dist bundle).

### Added
- `src/sandbox.cyr` — `mehman_sandbox_run_guest(spec, out_exit_code)`: validates
  the guest against the swallow-stage capability contract, then drives kavach's
  PROCESS-backend lifecycle (`kavach_init` → `config_new` →
  `config_backend(PROCESS)` → `sandbox_create` → `sandbox_transition(RUNNING)` →
  `sandbox_exec` → `sandbox_destroy`), mapping kavach failure signals onto the
  `MehmanError` namespace. **M1 acceptance met**: a trivial foreign binary
  (`/bin/true`, `/bin/false`) runs sandboxed and is reaped cleanly.
- `tests/mehman.tcyr` — M1 host group (spec rejection pre-init; real
  fork+exec+reap of `/bin/true`/`/bin/false`). **28 asserts total, all passing.**

### Changed
- Consume **kavach 3.6.0** via `[deps.kavach]` (git + `../kavach` path,
  `modules = ["dist/kavach.cyr"]`), source-included in `src/main.cyr`'s chain
  after `src/types.cyr`. `[deps].stdlib` expanded to kavach's full transitive
  set (required to link the bundle). See [ADR 0002](docs/adr/0002-consume-kavach-3.6.0-and-land-m1.md).

### Notes
- kavach's PROCESS backend reports a coarse exec status via `out_exit_code`
  (0 = ran + output captured, 1 = exec failed) — **not** the guest's own
  `WEXITSTATUS`. Documented in `src/sandbox.cyr`; true guest-exit propagation is
  a future kavach concern. M2 (foreign-surface capture) remains gated on
  aethersafha.
- Known-benign integration warnings from the heavy kavach bundle: `duplicate fn`
  (`syserr_*`/`err_*`, last-def-wins), a `lib/sandhi.cyr` arg-count warning on a
  credential path M1 does not exercise, and ~13 MB static scan tables.

## [0.1.0] — 2026-07-02

First release. Scaffold + the dependency-free foundation vocabulary the
swallow-stage modules build on. The M1 sandboxed foreign host is designed but
deferred to v0.2.0 (see Notes).

### Added
- Initial project scaffold (`cyrius init --lib --agent`, pin 6.3.5).
- Project identity: sovereign **compat / "swallow"-stage surface backend** for
  the AGNOS compositor (aethersafha) — hosts foreign-app surfaces as guests in a
  kavach sandbox. The legitimate successor to XWayland's *actual* job, done the
  sovereign way (run a foreign-ABI binary sandboxed, capture its surface) —
  **not** an X.Org port, **not** X11→Wayland translation. Post-MVP; `bhumi`
  (the platform backend) ships first.
- Architecture map in `src/main.cyr`: planned `sandbox` (kavach host) /
  `surface` (capture) / `shim` (per-ABI translation) / `guest` (lifecycle)
  modules.
- `src/types.cyr` — foundation vocabulary shared by every swallow-stage module,
  replacing "scaffold only" with a real (dependency-free) surface:
  - `MehmanError` enum + `mehman_err_name` / `mehman_err_print` — a single error
    namespace for all of mehman; diagnostics to stderr via the stdlib writer.
  - `MehmanCaps` bounded capability set + `caps_swallow_default` /
    `caps_is_swallow_valid` — models the swallow-stage contract (no native-
    protocol access; mediated I/O) as a checkable record, not a comment.
  - `MehmanGuestSpec` guest descriptor + `guest_spec_new` / `guest_spec_is_valid`
    — the minimal, validated input the M1 sandbox host will consume.
- `src/main.cyr` now source-includes `src/types.cyr` at the head of the module
  chain (ordered for single-pass forward-reference resolution). The
  `mehman_scaffold_ok` sentinel is retained (the roadmap removes it at M4).
- `tests/mehman.tcyr` — validation suite for the error namespace, capability
  contract (including rejection paths), and guest-spec validation (22 asserts
  total, up from 2).

### Notes
- M1 (`src/sandbox.cyr`, the kavach-sandboxed foreign host) is designed against
  kavach's `sandbox_create` / `sandbox_exec` / `sandbox_destroy` surface but is
  **deferred** to v0.2.0: kavach is not yet consumable as a Cyrius library
  dependency. See [ADR 0001](docs/adr/0001-defer-m1-until-kavach-is-consumable.md)
  and the M1 status note in [`docs/development/state.md`](docs/development/state.md).
