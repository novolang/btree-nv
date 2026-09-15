# btree-nv

A B+ tree is an ordered map held in fixed-size blocks, so that a
lookup reads a handful of blocks rather than the whole collection.
Douglas Comer's [The Ubiquitous B-Tree](https://dl.acm.org/doi/10.1145/356770.356776)
(ACM Computing Surveys, 1979) is the standard description, and
[SQLite's b-tree pages](https://www.sqlite.org/fileformat2.html#b_tree_pages)
are the working design this package follows. This package is the tree
and nothing else: it never opens a file, and it asks its caller for
every page it reads. [pager-nv](https://novo-lang.org/packages/pager-nv)
and [sql-engine-nv](https://novo-lang.org/packages/sql-engine-nv) are
built on it.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What it is

A **B+ tree** stores every key and every value in its bottom row of
nodes, called **leaves**. The nodes above, called **inner nodes**,
hold only **separator** keys and the addresses of the nodes below
them. Finding a key is a **descent**: read the root, compare the key
against its separators, read the child that range belongs to, and
repeat until a leaf. A descent therefore reads as many nodes as the
tree is deep, and no more.

Here one node occupies one **page**, a fixed-size block of bytes. The
key is an `Int`. The value is a **row**, a list of **cells**, and a
cell is one of five things: an integer, a float, text, a blob, or
NULL. Those are the five storage classes SQLite names, and nothing
else appears in a page.

**This package does not read the pages.** A descent cannot decide
which page it wants next until it has the page under the cursor, and
that read is the only part of a tree that touches a machine. So every
operation here is a **cursor** the caller advances one step at a time.
A step answers a row, or says the operation is finished, or says a
page did not decode, or asks the caller for a page and waits. The
caller performs the read and hands the bytes back.

The consequence for an end user is that one tree serves a database
file, a table held in memory and a test with a list of pages in it.
The difference between those three is entirely in whoever answers the
requests. Every function in this package performs no input or output,
and the compiler checks that on every build.

The page layout is this package's own, and it is given here in full
because a page written by one version and read by another is the same
bytes or it is nothing.

| Quantity | Value |
| --- | --- |
| Page size | 4096 bytes |
| Page header | 16 bytes |
| Payload after the header | 4080 bytes |
| Keys a node holds before it splits | 16 |
| Keys left in the left half after a split | 8 |
| Byte 0 of a leaf page | 6 |
| Byte 0 of an inner page | 7 |
| A key on the wire | 8 bytes, two's complement |

Every number in the format is big-endian, on disk and in the checksums
pager-nv computes over it.

**The page header**, the first 16 bytes of every page.

| Bytes | Contents |
| --- | --- |
| 0 | The page kind: 6 for a leaf, 7 for an inner node |
| 1 to 3 | Reserved, zero |
| 4 to 7 | The page id, unsigned 32-bit |
| 8 to 15 | Reserved, zero |

**A leaf payload**: a 32-bit entry count, then that many entries, then
zero padding to the end of the page. Each entry is an 8-byte key, a
32-bit encoded row length, and that many bytes of row.

**An inner payload**: a 32-bit child count K, then K page ids of 4
bytes each, then K-1 separator keys of 8 bytes each, then zero
padding. There is always one more child than separator.

**A cell** is a one-byte tag and its payload.

| Tag | Cell | Payload |
| --- | --- | --- |
| 0 | NULL | none |
| 1 | integer | 8 bytes, two's complement |
| 2 | float | 8 bytes, IEEE-754 binary64 |
| 3 | text | a 32-bit length, then that many UTF-8 bytes |
| 4 | blob | a 32-bit length, then that many raw bytes |

An encoded row is a 32-bit cell count, then each cell in order.

## Install

```
novo pkg add btree-nv
```

## Example

```novo
use btree
use nodefmt

// Where the pages come from. A real program reads them from a file or
// from flash; this one hands back a single empty leaf page.
fn read_page(page_id: Int) -> [Int]
    nodefmt.encode_header(nodefmt.KIND_LEAF, page_id)

fn main() [io]
    // A tree whose root is page 1, with no rows and one level. A pager
    // reads these three numbers out of the file header.
    let t = btree.open(1, 0, 1)

    // A cursor that will produce every row, in key order.
    var c = btree.scan(t)
    var rows = 0
    var going = true
    while going
        // One advance: the cursor comes back beside what it just did.
        let p = btree.step(c)
        c = p.cursor
        match p.step
            SRow(key, row) => rows = rows + 1
            SDone(n)       => going = false
            SFailed(e)     => going = false
            SRequest(q)    =>
                match q
                    // The tree wants a page. Read it and hand it back.
                    PrNeedPage(id)    => c = btree.feed_page(c, id, read_page(id))
                    // A scan never asks for these three.
                    PrWritePage(i, b) => going = false
                    PrAllocPage(k)    => going = false
                    PrFreePage(i)     => going = false
    println("${rows} rows")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: btree-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `btcell` | The five storage classes as one type, the accessors that read one, the byte encoding of a cell and of a row, and the SQL text of each. |
| `nodefmt` | One node as one page: the sizes, the two page kinds, the header, the leaf and inner encoders and decoders, and the binary search a descent uses. |
| `btree` | The tree as a state machine: the five operations, the cursor, the page request, and the two functions that feed an answer back. |
| `pageio` | The same walk as a function call: a trait with the host's four page operations, and two drivers that run a cursor to the end or to its next row. |

The module is `btcell` rather than `cell` because `cell` is a standard
library module name and a package may not ship one. The type is
`Cell`, and every function keeps the name it has.

## How to choose an entry point

**`btree.step` is the primitive.** The caller holds the loop, so it
can interleave two cursors, batch the reads several cursors are
waiting on, or put the cursor away and come back to it. Use it in a
program with an event loop, a batching reader or an asynchronous
pager.

**`pageio.run` is that loop written once.** Implement the `PageIo`
trait for whatever holds your pages and hand it to `run`, which
answers every request itself and comes back when the walk is over. Use
it when the page store is synchronous, which is what pager-nv is.
`pageio.next_row` is the same driver stopping at each row, for a query
that wants ten rows of a million.

`PageIo` carries one effect parameter, and `run` is charged exactly
what the implementation behind it costs (SPEC section 5.6). A pager
that reads a file supplies `impl PageIo[fs]` and the call costs
`[fs]`. A test that supplies a page table in memory implements
`PageIo[]` and the call costs nothing. Neither charge lands on this
package.

## The rules a user needs

1. **Every operation is a cursor, and `step` answers one of four
   things.** `SRow` is a row and its key. `SDone` is the operation
   finished, carrying the rows it affected. `SFailed` is a page that
   did not decode. `SRequest` is the tree asking, and the caller
   answers it with `feed_page` or `feed_alloc` and steps again.
2. **`feed_page` takes a whole page, header included**, exactly
   `nodefmt.page_size()` bytes long. The header's page id is checked
   against the id that was asked for, and a disagreement finishes the
   cursor with `IdMismatch`. That check is the cheapest detector there
   is for a read that went to the wrong offset.
3. **Keep the tree `btree.tree_of` answers after an insert.** An
   insert that filled the root splits it, and the root moves to a new
   page. The caller stores the new root id wherever the next `open`
   will read it.
4. **Deleting a key that is not there is not an error.** The cursor
   finishes `SDone(0)`.
5. **A cursor that has finished stays finished.** Stepping it again
   answers `SFailed(Finished)`, and a cursor that has failed repeats
   its failure on every later step.
6. **Read a cell with the accessors, never by matching `Cell`.**
   `as_int`, `as_float`, `as_str`, `as_blob` and `is_null` each answer
   `None` when the cell holds something else. A sixth storage class
   would then be one new arm here rather than one in every `match` in
   the calling program.
7. **A row must fit one page on its own.** A row larger than the 4080
   bytes of payload is refused with `RowTooLarge`, naming the key, the
   bytes needed and the bytes available. There is no overflow-page
   chain in this version.
8. **`btcell.tag` is also a sort key.** The tag numbers in the cell
   table above are the order SQL comparison uses when it has to order
   values of different storage classes.
9. **Nothing here is mutated.** A cursor comes back from `step` rather
   than being changed in place, so a caller may hold two cursors over
   one tree and know they cannot alias (SPEC section 14).
10. **`btree.create` answers a request, not a tree.** A tree with no
    root page is not a tree, so an empty tree is made by performing
    that one allocation and passing the id to `btree.open`.
11. **The tree never decides what freeing a page means.**
    `PrFreePage` says a page is no longer reachable. Whether that is
    reuse now or reuse at the next commit is the page store's
    decision.

## Running on a microcontroller

The package declares no effects, and a program that calls it links for
a device target. `novo build --target=nrf52-qemu` builds and links a
program calling `nodefmt.page_size()`.

A program that carries a page cannot be built for that tier today.
Every function that takes or returns page bytes takes a `[Int]`, and
the embedded tier refuses a list literal as an implicit heap
allocation, with error E4000. A fixed-capacity page buffer is what
would lift that, and it is not in this release.

## What is not included

- **Any input or output.** No file is opened, no page is read and no
  page is written. `PageRequest` is how the tree asks, and
  [pager-nv](https://novo-lang.org/packages/pager-nv) is what answers.
- **Overflow pages.** A row that does not fit one page is refused
  rather than split across a chain. See rule 7.
- **Keys other than `Int`.** A tree keyed by text needs a comparison
  function in the node format, and that is a format change rather than
  an addition.
- **Transactions, locking and crash recovery.** Those belong to
  whatever owns the file. pager-nv has the write-ahead log.
- **A free list.** The tree says a page is free and stops there. See
  rule 11.
- **A configurable page size.** `nodefmt.page_size()` is 4096 in this
  version. It is a function rather than a constant so that a later
  version can read it from a file header without any call site
  changing.

## Related packages

- [pager-nv](https://novo-lang.org/packages/pager-nv) is the other
  half: the page file, its header, its write-ahead log and the
  checksums over it. It answers the requests this package makes, as an
  `impl PageIo[fs]`.
- [sql-engine-nv](https://novo-lang.org/packages/sql-engine-nv) is SQL
  over this tree. Its executor relays the page requests its cursors
  make, so a host drives one loop rather than two.
- [sqlite-nv](https://novo-lang.org/packages/sqlite-nv) reads and
  writes a real SQLite database file. Take that package to open a file
  some other program wrote. Take this one for a tree in a format of
  your own.
- [lsm-nv](https://novo-lang.org/packages/lsm-nv) is the other storage
  shape: writes are appended and merged later, rather than updated in
  a page in place. It suits a write-heavy store on flash.
- `std.collections` in the standard library has `OrderedMap`, which is
  an ordered map in memory with no pages and no file under it. Take it
  when the whole collection fits in memory and nothing has to survive
  the process.

## Tests

```bash
novo test tests/btcell_tests.nv    #  9 tests: the cells and their bytes
novo test tests/nodefmt_tests.nv   # 10 tests: the page layout, by offset
novo test tests/btree_tests.nv     # 12 tests: the request protocol, driven by hand
novo test tests/pageio_tests.nv    #  5 tests: the driver and what it costs
```

The reference data is the format above. `nodefmt_tests.nv` asserts the
byte offsets one at a time, because a page written by one version and
read by another is the format or it is nothing. `btree_tests.nv` is
written the way a host writes: a loop that steps a cursor, answers
what it asks for, and steps again. `pageio_tests.nv` supplies a page
table in memory as an `impl PageIo[]` and checks that a walk over it
is charged no effect, which the compiler decides before the run
starts.

The tests compile today and fail at run, each on the
`not implemented: btree-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at
a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `nodefmt.KIND_LEAF`, `.KIND_INNER` | yes (they are constants) |
| `btcell.of_int`, `.of_float`, `.of_str`, `.of_blob`, `.null_cell` | no |
| `btcell.as_int`, `.as_float`, `.as_str`, `.as_blob`, `.is_null`, `.tag` | no |
| `btcell.encoded_size`, `.encoded_row_size` | no |
| `btcell.encode_cell`, `.encode_row`, `.decode_cell`, `.decode_row` | no |
| `btcell.show`, `.show_row`, `CellError.message` | no |
| `nodefmt.page_size`, `.header_size`, `.payload_size`, `.fanout`, `.split_at` | no |
| `nodefmt.encode_header`, `.kind_of`, `.id_of` | no |
| `nodefmt.encode_leaf`, `.encode_inner`, `.decode_leaf`, `.decode_inner` | no |
| `nodefmt.leaf_encoded_size`, `.inner_encoded_size`, `.seek`, `.child_for` | no |
| `nodefmt.NodeError.message` | no |
| `btree.open`, `.create`, `.lookup`, `.scan`, `.range`, `.insert`, `.delete` | no |
| `btree.step`, `.feed_page`, `.feed_alloc`, `.awaiting`, `.tree_of` | no |
| `btree.BtreeError.message` | no |
| `pageio.run`, `.next_row`, `IoFault.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
