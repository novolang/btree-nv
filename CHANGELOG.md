# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

## 0.0.3 — 2026-09-12

- **BREAKING — the `cell` module is now `btcell`.**  `cell` is a standard
  library module name, and a package may not ship one: the build refuses
  `src/cell.nv`, so 0.0.2 cannot be installed at all.  The type is still
  `Cell` and every function keeps its name and its signature — only the
  module moved, so a consumer replaces `use cell` with `use btcell` and
  `cell.` with `btcell.`.  The break is a patch rather than a minor
  because 0.0.2 builds for nobody: there is no working consumer to break.

## 0.0.2 — 2026-09-09

- **Toolchain floor is 0.8.9**: the bodies and signatures use what 0.8.9 added (`todo()`, a bound effect parameter, the four layers), and the manifest says so instead of letting an older toolchain fail on an undefined function.  No signature changed.

## 0.0.1 — 2026-09-09

- First interface release: every public signature and effect row, every body a `todo()`.  **NOT IMPLEMENTED — interface only.**
