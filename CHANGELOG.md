# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.1.0] — 2026-06-29

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
