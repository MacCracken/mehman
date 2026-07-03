# mehman — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.1.0** — first release, cut 2026-07-02. Scaffold (`cyrius init`, 2026-06-29)
plus the dependency-free **foundation vocabulary** (`src/types.cyr`) that the
swallow-stage modules build on. M1 (the sandboxed foreign host, v0.2.0) is
designed but deferred — see the M1 status note below.

## Toolchain

- **Cyrius pin**: `6.3.5` (in `cyrius.cyml [package].cyrius`; wrapper is 6.3.35 —
  benign drift, warned on every build).

## Source

- `src/main.cyr` — library header / module-include chain. Includes
  `src/types.cyr`; retains the `mehman_scaffold_ok` sentinel (removed at M4).
- `src/types.cyr` — **foundation vocabulary** (dependency-free):
  - `MehmanError` enum + `mehman_err_name` / `mehman_err_print`.
  - `MehmanCaps` (bounded capability set) + `caps_swallow_default` /
    `caps_is_swallow_valid` — the swallow-stage contract (no native-protocol
    access; mediated I/O) as a validatable record.
  - `MehmanGuestSpec` (guest descriptor) + `guest_spec_new` /
    `guest_spec_is_valid` — the validated input the M1 host consumes.

## M1 status — designed, blocked on kavach consumability

The M1 host (`src/sandbox.cyr`, `mehman_sandbox_run_guest`) is fully designed
against kavach 3.5.4's real surface (`kavach_init` → `config_new` →
`config_backend(PROCESS)` → `sandbox_create` → `sandbox_transition(RUNNING)` →
`sandbox_exec` → read `ExecResult` → `sandbox_destroy`). It is **not yet
implementable** because kavach cannot currently be consumed as a Cyrius library:

- kavach has **no `dist/kavach.cyr` bundle** and **no `[lib]` section**, and this
  toolchain has **no bundle generator** (`cyrius distlib` does not exist;
  `cyrius lib` only syncs stdlib; `cyrius package` produces a `.ark`).
- kavach's `src/main.cyr` is a **program entry point** (its own `main()` + demo +
  `syscall(SYS_EXIT)`) that pulls 40+ siblings via relative `include`s. Resolving
  it as a dep (`[deps.kavach] modules=["src/main.cyr"]`) materializes only that
  one file, so the build fails: `error: cannot open include file: src/util.cyr`.

Unblocking M1 requires making kavach consumable (a library aggregation header +
committed `dist/kavach.cyr` bundle in the kavach repo). That is kavach-repo work;
tracked as the M1 gate.

## Tests

- `tests/mehman.tcyr` — primary suite: smoke + foundation validation
  (error namespace, capability contract incl. rejection paths, guest-spec
  validation). **22 asserts, all passing** on `cyrius test`.
- `tests/mehman.bcyr` — benchmark stub (no-op)
- `tests/mehman.fcyr` — fuzz stub

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert

## Consumers

Intended: **aethersafha** (compositor) — sources foreign-app surfaces from
mehman. None wired yet (post-MVP / swallow stage; `bhumi` ships first).

## Next

See [`roadmap.md`](roadmap.md).
