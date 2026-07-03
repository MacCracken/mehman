# 0005 — M3 protocol shim: a per-ABI event wire, with delivery deferred

**Status**: Accepted
**Date**: 2026-07-03

## Context

M3 is the protocol shim: per-foreign-ABI translation of input / resize / lifecycle
events between the compositor and the guest. Its acceptance is *"a guest receives
input and resizes correctly."* The compositor side (aethersafha) already speaks a
concrete event vocabulary — USB **HID usage** keycodes (`HID_ESC = 0x29`…) mapped
to actions, `win_resize(w, h)`, and a `WinState` lifecycle (normal / minimized /
maximized / …).

Two facts shape the design:

1. mehman **owns** the compositor↔guest contract (aethersafha imports mehman's
   types), so the event vocabulary and its wire encoding belong here.
2. A hosted guest runs under kavach's **PROCESS backend**, which is **one-shot**:
   `fork + exec + waitpid`, capture stdout, reap. There is **no live stdin** on a
   running guest, so there is nothing to stream events *to* yet — the same shape
   that deferred M2's real-pixel capture behind the stdout MVP.

## Decision

Land the M3 **foundation** in `src/shim.cyr`, dependency-free: the event
vocabulary (`MehmanInputEvent` + `MehmanInputKind` {key, pointer-button,
pointer-motion}, `MehmanLifecycle`, `MehmanAbi`) and the per-ABI **encoders** that
translate an event into a compact byte wire —

- input: `[0x01] key`, `[0x02] pointer-button`, `[0x03] pointer-motion` (u16-LE
  x/y); resize: `[0x10] w:u16-LE h:u16-LE`; lifecycle: `[0x20] kind`.

`MehmanAbi` has one case today (`ABI_SWALLOW`); it is the extension point — a new
foreign ABI adds a case and its own encoders (that is what "per-foreign-ABI"
means). Encoders return the byte count, or 0 for a null event / unsupported ABI.

**Defer live delivery.** Writing the encoded bytes to a running guest is out of
scope until a persistent-guest execution model exists; the translation (the hard,
contract-defining part) ships now and is fully testable.

## Consequences

- **Positive** — the compositor↔guest event contract is real, byte-tested, and
  dependency-free, so aethersafha can build events and translate them without
  pulling the sandbox/kavach surface. The wire is stable behind a small API, so
  fidelity/delivery can evolve without breaking callers.
- **Negative** — M3's acceptance is not met: no guest actually *receives* input or
  a resize yet. The wire is a contract, not a delivered channel.
- **Neutral** — delivery needs a persistent-guest model (a kavach change): a guest
  that stays alive with a readable input channel (stdin or a shared queue). Resize
  additionally implies re-sizing the `MehmanSurface` buffer (a surface concern),
  and richer input (key repeat, modifiers, scroll) will extend the wire.

## Alternatives considered

- **Block M3 until a persistent guest exists.** Rejected — the translation is the
  contract-defining, testable half and is what a consumer needs to build against;
  gating it on execution-model work stalls the seam for no benefit.
- **Reuse the compositor's raw HID/WinState values as the wire** (no mehman
  vocabulary). Rejected — it couples the guest wire to aethersafha's internals;
  mehman owns a stable, ABI-neutral contract that the compositor encodes *into*.
- **Encode to text (like the stdout capture MVP).** Rejected — events are small,
  structured, and high-frequency; a compact tagged binary wire is the right shape
  and still trivial to test.
