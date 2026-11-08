# 0009 — Confine live guests, and survive their exit

**Status**: Accepted
**Date**: 2026-11-08

## Context

The roadmap's boundary is that a foreign app is *always* kavach-sandboxed. ADR 0006 built the live path on kavach's
`persistent_spawn`, which applies no seccomp, no landlock and no capability bound. kavach's own doc says so, and it added
`persistent_spawn_confined` for payloads that are not trusted. mehman never moved. A one-shot guest ran under
`policy_basic()`, and a live guest ran with the caller's full authority. Measured through `guest_start` at v1.1.1:
`NoNewPrivs` 0, `Seccomp` 0, and a TCP connect to a local listener succeeded. The 2026-07-03 audit did not catch it: its
finding 1 covers the pre-exec command checks, which both paths have.

Separately, a write to the stdin of a guest that has exited raises SIGPIPE, whose default action kills the writer. The
audit recorded this as Low and recommended ignoring SIGPIPE once the stdlib could. Measured at v1.1.1: the next event sent
to an exited guest ended the host with status 141. A guest decides when it exits, so a foreign binary could end the
compositor.

The API is frozen at v1.0: additive changes only within 1.x.

## Decision

**`guest_start` spawns through `persistent_spawn_confined(command, policy_basic())`**, after `kavach_init`, which builds
the tables the filter is made from. `policy_basic()` is what a one-shot guest gets from `config_new()`, so both paths now
run one policy.

**`guest_send_input` and `guest_send_resize` block SIGPIPE on the calling thread for the write**, take it with a
zero-timeout `rt_sigtimedwait` if the write returned EPIPE, restore the caller's mask, and return `EXEC_FAILED`. A SIGPIPE
the caller already had blocked is left to the caller. This is the pattern `lib/tls.cyr` uses for the same reason.

## Consequences

- **Positive** — the boundary holds on both paths, and a guest cannot end the host by exiting. kavach's 3.14.8 network
  denial and landlock domain reach live guests, which they could not on the unconfined path.
- **Negative** — a live guest that relied on the network, or on a syscall the filter denies, stops working. This is a
  behaviour change inside a frozen API, accepted because the old behaviour broke the stated boundary. It is recorded in
  `docs/api.md`.
- **Negative** — `guest_send_*` now return `EXEC_FAILED` for a guest that has gone, where they returned `OK`. A caller
  that ignored the return is unaffected.
- **Negative** — each send costs two more syscalls, on a path whose write is already one.
- **Neutral** — known limits, measured at v1.1.2 and on the roadmap. `policy_basic()` names no filesystem rule, so a guest
  reads what its uid can. A guest that stops reading its stdin holds up a send once the 64 KiB pipe is full, until the
  guest exits. The fix here is what makes that send return at all.

## Alternatives considered

- **`signal_ignore(SIGPIPE)`,** which the audit and ADR 0006 recommended. It is three lines. But SIG_IGN is process-wide,
  overrides a consumer's own handling, and survives `execve`, so every guest started afterwards would inherit it: `seq`
  past the output cap would see EPIPE where it dies with 141 today.
- **`policy_strict()`.** Its read-only root needs a rootfs, and its resource limits need a cgroup the persistent path does
  not set up, so most of what it adds would be promised and not applied. The two guest paths would also differ.
- **Require namespaces (`persistent_spawn_confined_ns`).** It fails closed where a host refuses unprivileged user
  namespaces, as Ubuntu 24.04 does. Guests that run today would stop running, and the one-shot path does not require them.
- **Check whether the guest is alive before writing.** Racy: it can exit between the check and the write, and a `waitpid`
  here would reap the child `persistent_terminate` expects.
- **Fix it in kavach's `persistent_send`.** The better long-term home, since every consumer has the hazard. It would not
  have closed it for this release, and the mehman side is a single private function to delete when it lands.
