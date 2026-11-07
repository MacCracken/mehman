# 0008 — Report a deadline kill as its own error

**Status**: Accepted
**Date**: 2026-11-06

## Context

mehman sets no timeout on a one-shot guest, so it gets kavach's default of 30 s. Through v1.1.0 a guest killed at
that deadline returned `MehmanError.OK`: `_mehman_run_guest` read the exit status and ignored `ExecResult.timed_out`.

That mattered more at v1.1.1. Up to kavach 3.14.0 a guest could outrun the deadline by closing its output, which
ran to completion and read as a clean success. kavach 3.14.4 closed that, so the same guest is now killed at 30 s, and
mehman reported the kill as `OK` with `out_exit_code` 137. A caller branching on the return, as `docs/api.md` told it
to, could not tell a guest that finished from one that was killed.

The API is frozen at v1.0: additive changes only within 1.x.

## Decision

`MehmanError` gains `GUEST_TIMED_OUT` (7), appended so no existing value moves. `_mehman_run_guest` returns it when
kavach reports `timed_out`, after the guest is reaped. What the run did is still reported: `out_exit_code` is 137, and
output written before the kill reaches the surface.

- A failed reap outranks the timeout, since `GUEST_REAP_FAILED` says something is still wrong.
- `guest_run` records the status and does not move to `RUNNING`, because the guest has been reaped. The guest stays in
  its state and may be run again or evicted.
- kavach's externalization gate runs first. A killed guest whose output is quarantined returns `EXEC_BLOCKED` with
  `out_exit_code` untouched, because kavach returns no result to read.
- `guest_start` guests have no deadline and are unchanged.

`_mehman_deadline_ms` is a private test seam, 0 for kavach's default. It is not API. It exists so the suite can run
this path against real guests in 200 ms, where a 30 s deadline would keep it out of the suite.

## Consequences

- **Positive** — the return alone says whether a guest finished. The kill path is tested for the first time.
- **Negative** — a caller that treats every non-`OK` return as "the guest never ran" now sees a guest that ran and was
  killed. This is a behaviour change inside a frozen API, accepted because the old `OK` was misleading, and recorded
  in `docs/api.md`.
- **Negative** — a known limit, in kavach: `confine_last_timed_out` is only written at the end of a capture. A `fork`
  failure on the run straight after a timed-out run returns early, so that run reads the stale flag and reports
  `GUEST_TIMED_OUT`. mehman cannot tell the two apart, and the case needs `fork` itself to fail.
- **Neutral** — callers still cannot choose a deadline. A public timeout is its own decision.

## Alternatives considered

- **Keep `OK` and document 137.** v1.1.1's first draft did this. It leaves every caller to decode a signal number, and
  137 only says SIGKILL, not who sent it. kavach already reports the fact directly.
- **Reuse `EXEC_FAILED`.** The guest did execute, and it may have produced output, so that name would mislead.
- **Move a timed-out guest to `RUNNING`.** That is what a successful `guest_run` does, and what `OK` used to cause, but
  nothing is running.
