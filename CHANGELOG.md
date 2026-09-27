# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

## 0.1.0 — 2026-09-27

The first implementation of the interface published as 0.0.1: the
cells and their bytes, the node format, the tree as a page-request
state machine, and the drivers over a page store.

### Behaviour the interface left open

- A page shorter than 4096 bytes reads as though padded with zeros, so
  a 16-byte leaf header is an empty leaf.  A longer page is refused
  with `BadPageSize`.
- An insert or a delete plans every page it changes when it reaches its
  leaf, then asks for the allocations, the writes from the leaf up with
  a new root last, and the frees, in that order.  A new page is named in
  the plan by a negative number until `feed_alloc` answers it.
- A leaf one key over the limit splits 8 and 9; a leaf whose rows no
  longer fit a page is split by size into as many leaves as it takes.
  An inner node over the limit gives up its separator after 8.
- A delete that empties a leaf frees it and removes it from its parent,
  an inner node left with no child is freed too, and a root left with
  one child gives way to it.  Nodes are not merged below half full.
- `feed_page` and `feed_alloc` fail a cursor with `Unexpected` when it
  was not waiting for that page or for an allocation; a page met twice
  on one path, or a path past 64 pages, is `Cycle`.
- A lookup produces its row and then `SDone(1)`.  A scan that reaches
  its end sets the tree's size.
- `show` renders a float as novo-lang prints it and doubles a quote
  inside text; `decode_cell` checks text for UTF-8.

### Changes to the interface

- `Cursor` gains the fields its state needs: `nodes`, `started`,
  `applying`, `allocs`, `fresh`, `writes`, `frees` and `next`, with the
  new types `BtNode` and `BtWrite`.  The fields that change are `var`,
  and every function answers a new cursor and leaves its argument as
  it was.
- The README's device claim is withdrawn.  A page is a list, which the
  embedded tier refuses.

### Toolchain

- The toolchain floor is 0.13.0.

### Tests

- 44 tests in four suites, with 100% line coverage over `src/`
  measured by `tests/coverage.sh`.  `btree_tests.nv` runs random
  inserts and deletes against a sorted-list model and checks the whole
  tree's invariants after every operation.

## 0.0.4 — 2026-09-15

- README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

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
