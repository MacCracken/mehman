# 0006 — M3 live event delivery via a kavach persistent-guest model

**Status**: Accepted
**Date**: 2026-07-03

## Context

[ADR 0005](0005-m3-shim-event-wire-and-deferred-delivery.md) shipped the M3
translation (events → a byte wire) but **deferred live delivery**: a kavach
PROCESS-backend guest was one-shot (`fork` + `exec` + `waitpid`, capture stdout,
reap), so there was no running guest with a live input channel to stream events
to. M3's acceptance — *"a guest receives input and resizes correctly"* — could not
be met without an execution-model change in kavach.

## Decision

Add a **persistent-guest execution model to kavach (3.7.0)** and wire mehman's M3
delivery on top of it.

**kavach `src/persistent.cyr`**: `persistent_spawn(command)` runs the same pre-exec
safety checks the one-shot path enforces (`is_safe_argument` + `check_command`),
then `fork`s with **two pipes** — the parent writes the guest's stdin, reads its
stdout — `dup2`s them onto fd 0/1 and `execve`s. The guest stays alive.
`persistent_send` / `persistent_read` stream to/from it; `persistent_terminate`
closes stdin (EOF), `SIGKILL`s, and `waitpid`s to reap. A `PersistentGuest` handle
holds `{pid, stdin_fd, stdout_fd, alive}`.

**mehman `src/guest.cyr`**: `MehmanGuest` gains a live `handle`. `guest_start`
spawns the persistent guest; `guest_send_input` / `guest_send_resize` shim-encode a
compositor event and `persistent_send` its wire bytes to the guest's stdin;
`guest_read` reads its output; `guest_evict` terminates + reaps the handle. This is
additive — the one-shot `guest_run` / `mehman_sandbox_capture_guest` path is
unchanged.

## Consequences

- **Positive** — M3's acceptance is met, verified end-to-end: a shim-encoded input
  event is delivered to a running `/bin/cat` guest and read back byte-for-byte
  (`[0x01,0x29,1]`). The swallow stage now spans host → sandbox → capture →
  **live event delivery**. Roadmap M1–M4 are all complete.
- **Negative** — persistent streaming is **not** externalization-gate-scanned
  (the gate scans a full captured buffer, meaningless for a stream); persistent
  guests are for trusted-shape guests behind the capability contract (audited
  2026-07-03, finding #4). Writing to a self-exited guest can raise SIGPIPE
  (no `SIG_IGN` wrapper yet; `send` guards on the `alive` flag). Reads are blocking.
- **Neutral** — follow-ons: a non-blocking `guest_read` (`O_NONBLOCK`/`pipe2`) for a
  compositor frame loop; SIGPIPE ignore when a `sigaction` stdlib wrapper lands; an
  incremental streaming scanner for untrusted persistent guests.

## Alternatives considered

- **Feed initial input at one-shot launch** (stdin-at-exec, no persistence).
  Rejected — not *live*; a resize or keypress after launch could not be delivered,
  so it would not meet the acceptance.
- **A shared-memory event queue** instead of a stdin pipe. Deferred — pipes reuse
  the existing capture/exec machinery and syscalls, are trivially testable
  (`/bin/cat`), and match the stdout-as-framebuffer MVP's shape; a shm ring is the
  natural upgrade alongside real-pixel capture.
- **Keep M3 delivery deferred until a compositor needs it.** Rejected — aethersafha
  is wiring foreign-app interaction now, and this is the last swallow-stage gap
  before v1.0.
