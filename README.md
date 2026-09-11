# btree-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package
works; calling it panics with `not implemented`.

## What this is

An ordered map from `Int` to a row of values, stored one B+-tree node
per fixed-size page — and **it does not read the pages**.  A lookup is
a descent, a descent needs the page under the cursor before it can
decide which page it wants next, and that read is the only thing in a
tree that touches the machine.  So it is the only thing this package
refuses to do: every operation is a cursor the caller advances, and
when the cursor needs a page it says so and waits.

For an end user that means one B+-tree serves a database file, a
firmware image with a tree in flash, and a test with a list of pages in
memory — because the difference between those three is entirely in
whoever answers `PrNeedPage`, and never in the tree.

## The one example that will work

```novo
use std.list
use btree
use btcell

// Read every row of a tree, answering each page request from a store
// the caller already has.  `fetch` is the host's; nothing else here is.
fn dump(t: btree.Tree) -> [[btcell.Cell]] [fs]
    var c = btree.scan(t)
    var rows: [[btcell.Cell]] = []
    var going = true
    while going
        let p = btree.step(c)
        c = p.cursor
        match p.step
            SRow(k, r)  => rows = list.push(rows, r)
            SDone(n)    => going = false
            SFailed(e)  => going = false
            SRequest(q) =>
                match q
                    PrNeedPage(id)    => c = btree.feed_page(c, id, fetch(id))
                    // A scan asks for none of these; stopping is what
                    // a host does when the tree asks for something the
                    // operation it started cannot need.
                    PrWritePage(i, b) => going = false
                    PrAllocPage(k)    => going = false
                    PrFreePage(i)     => going = false
    rows
```

The `[fs]` on that function is the caller's, from its own `fetch`.
Nothing in btree-nv contributed to it.

The value module is `btcell` and not `cell` because `cell` is a standard
library module name, and a package may not ship one — the build refuses
the file before it reads it.  Only the module name is affected: the type
is `Cell`, and every function keeps the name it had.

## The layer, and why

`core`.  The budget is `[]` and the package keeps it by construction:
every function takes bytes the caller already holds and returns bytes
the caller will store.  There is no file handle anywhere in the
surface, no clock, no allocator hook — a `Tree` is three integers and
a `Cursor` is a path and a slot per level.

That is what lets the same tree compile into firmware.  It is also why
widening any function here later would be a breaking change: an
embedded consumer of `0.1` would stop compiling on a `0.2` that
reached for `[fs]`, which is exactly the promise the layer is making.

## The load-bearing interface

```novo
pub enum PageRequest
    PrNeedPage(page_id: Int)
    PrWritePage(page_id: Int, bytes: [Int])
    PrAllocPage(kind: Int)
    PrFreePage(page_id: Int)
```

**This package owns `PageRequest`, and sql-engine-nv depends on
btree-nv to get it.**  Both are `core`, so either direction satisfies
the layer rule and the choice had to be made on what the type means.
It means *the tree ran out of pages*: the SQL engine has no page of its
own to want, and every request that leaves `sql-engine-nv.step` came
from a cursor this package handed it.  A type that flows downward
through a relay belongs at the bottom of the relay, so it is declared
here and sql-engine-nv re-exports it in `StepResult`.

The consequence worth stating before anyone implements a body: the
must-have plan has sql-engine-nv at **P0** and btree-nv at **P1**, and
this dependency inverts that — the P0 package cannot be implemented
before its P1 dependency has a page format.  The interface milestone
does not care, because neither has a body.  The implementation order
does, and this is where it is written down.

`btree.KeyedRow` is the same shape from the caller's side: `SRow`,
`SDone`, `SRequest(PageRequest)`, `SFailed`.  Four arms, and the
protocol is the whole of it.

The second shape, in `pageio`, is the same walk as a function call
instead of a protocol — `pub fn run<S: PageIo[e]>(…) [e]`, charged
what the caller's own `PageIo` impl costs (SPEC § 5.6).  Use `step`
when the host has to interleave, batch or suspend; use `run` when the
pager is synchronous.  **`run` does not compile against an impl in
another module today**, and not because of anything in this package: a
toolchain defect drops the bound when it specialises the function, so
the clone is checked with `e` read as a typo'd effect label.  It is
filed against novo-lang as
*cross-module-specialisation-of-an-effect-polymorphic-function-drops-its-bound*.
The signature is published anyway, because it is the design this
release exists to have reviewed and the defect is the compiler's.

## The reference implementation

SQLite's b-tree layer (`btree.c`), as the design, and **novodb's
`page_btree.nv` and `page.nv` as the code**, which is what this
package is extracted from — the 4096-byte page, the 16-byte header,
the big-endian entry format, the fanout of 16.

Three things did not come across unchanged, and each is a decision
rather than an omission:

| novodb | here | why |
| --- | --- | --- |
| `lazy_btree_get` and friends declare `[fs]` and take a `PageSource` that reads the file | `btree.step` declares nothing and returns `PrNeedPage` | the walk is the same; the read leaves with it, which is what `core` means |
| the paged tree is **read-only**; writes go through a whole-file rewrite in `pagefile.page_log_save` | `PrWritePage`, `PrAllocPage` and `PrFreePage` | there was nothing to port, so the write half is designed here |
| a float cell encodes as the TEXT of `string_of_float`, because the stdlib had no `float.to_bits` | 8 bytes, IEEE-754 binary64, big-endian | the text form is neither compact nor round-trip exact |

The fanout of 16 is measured and not inherited: over M in {4, 8, 16,
32} on novodb's 1000-row benchmark every read path roughly halved from
4 to 16 while insert stayed inside noise, and 32 started costing
inserts.

## Building it, and checking it

```bash
novo pkg add btree-nv          # add it to a package
novo pkg build                 # type-check and effect-check every module
novo test tests/btree_tests.nv # the API tests — see below
```

**`novo test` is red on every suite, and that is the published state.**
Each test calls a function whose body is `todo()`, so the first
assertion in each file panics with `not implemented: btree-nv.…`:

```
$ novo test tests/btcell_tests.nv
  ✗ test_constructors_and_accessors_round_trip
      not implemented: btree-nv.btcell.of_int
  0 passed, 1 failed
```

The tests are the design under review, not a regression net.  When the
bodies land they become the first real assertions, unchanged.

## Status

| function | implemented |
| --- | --- |
| `btcell.of_int`, `of_float`, `of_str`, `of_blob`, `null_cell` | no |
| `btcell.as_int`, `as_float`, `as_str`, `as_blob`, `is_null`, `tag` | no |
| `btcell.encoded_size`, `encoded_row_size` | no |
| `btcell.encode_cell`, `encode_row`, `decode_cell`, `decode_row` | no |
| `btcell.show`, `show_row` | no |
| `nodefmt.page_size`, `header_size`, `payload_size`, `fanout`, `split_at` | no |
| `nodefmt.encode_header`, `kind_of`, `id_of` | no |
| `nodefmt.encode_leaf`, `encode_inner`, `decode_leaf`, `decode_inner` | no |
| `nodefmt.leaf_encoded_size`, `inner_encoded_size`, `seek`, `child_for` | no |
| `btree.open`, `create`, `lookup`, `scan`, `range`, `insert`, `delete` | no |
| `btree.step`, `feed_page`, `feed_alloc`, `awaiting`, `tree_of` | no |
| `pageio.run`, `next_row` | no |
