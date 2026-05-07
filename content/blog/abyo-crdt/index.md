+++
title = "abyo-crdt: a pure-Rust CRDT library that's 17× faster than yrs at append-heavy workloads"
date = 2026-05-08
description = "abyo-crdt is a Pure Rust CRDT library combining Fugue-Maximal lists, Peritext rich text, and an order-statistic AVL tree for O(log N) operations across the board. 167 tests, 3.3M+ fuzz runs, exhaustive stateright model checking, Yjs/Quill Delta interop, WASM + Python bindings. This post walks through the algorithm choices, the AVL bug adversarial fuzz caught before release, and the head-to-head numbers vs yrs."

[taxonomies]
tags = ["rust", "launch", "library", "crdt", "algorithm", "local-first"]

[extra]
toc = true
+++

Local-first software has been waiting for a Rust-native CRDT library
that doesn't apologize for its host language. The mature options today
either treat Rust as the implementation detail behind a JavaScript or
Python API ([yrs] is the Rust port of Yjs, [automerge] has a Rust
core but the JS bindings are the priority), or they're explicitly
research prototypes ([diamond-types] is Joseph Gentle's Eg-walker
reference). If you want to drop a CRDT into a Tauri app, a Bevy
multiplayer game, an embedded device, or — in our case — a Rust
backend serving a Yjs browser client, you're either accepting a WASM
round-trip or rebuilding parts of the library yourself.

Today I'm releasing `abyo-crdt`: a Pure Rust CRDT library with first-
class native APIs, built around the most recent algorithmic research —
[Eg-walker (Gentle, 2024)][eg-walker], [Fugue-Maximal (Weidner et al.,
2023)][fugue], and [Peritext (Litt et al., 2022)][peritext] — with the
production-grade engineering that the abyo software brand is staking
its reputation on.

[yrs]: https://github.com/y-crdt/y-crdt
[automerge]: https://github.com/automerge/automerge
[diamond-types]: https://github.com/josephg/diamond-types
[eg-walker]: https://arxiv.org/abs/2409.14252
[fugue]: https://arxiv.org/abs/2305.00583
[peritext]: https://www.inkandswitch.com/peritext/

This post walks through what's in the box, the algorithm story, the
performance numbers, and one real bug I found via adversarial fuzz
testing before the v0.4.0-alpha.1 release.

## The headline

Single-threaded scalar code, criterion bench on AMD Ryzen 9 9950X,
Linux, rustc 1.95.0 stable, `--release`. Bench source:
[`benches/vs_yrs.rs`][vs-yrs]. `yrs` v0.23.0 is the Rust port of Yjs;
both libraries are inserting one character at a time into an empty
text document.

| size  | abyo-crdt | yrs       | abyo : yrs |
| ----: | --------: | --------: | --------- |
| 100   | 45 µs     | 38 µs     | 0.84×     |
| 1,000 | 494 µs    | 2.26 ms   | **4.6×**  |
| 5,000 | 3.1 ms    | 53 ms     | **17×**   |

[vs-yrs]: https://github.com/abyo-software/abyo-crdt/blob/main/benches/vs_yrs.rs

The crossover happens around 100 characters because `yrs` carries
fixed per-op overhead (struct-store bookkeeping, awareness, undo
manager scaffolding) that beats abyo-crdt's slightly heavier
algorithmic work at small sizes. Past a few hundred chars our
asymptotic behavior wins: abyo-crdt's order-statistic AVL tree gives
`O(log N)` for every list operation including remote inserts and
`OpId → position` lookups, where `yrs` does more bookkeeping per insert
and pays for its richer feature set.

## What's in the box

Five CRDT data types, a binary persistence layer, two FFI crates,
production-grade verification:

| Component                              | Status                                   |
| -------------------------------------- | ---------------------------------------- |
| `List<T>` — Fugue-Maximal sequence     | ✅ AVL OST, `O(log N)` for all ops       |
| `Map<K, V>` — LWW-Map                  | ✅                                       |
| `Counter` — PN-Counter                 | ✅                                       |
| `Set<T>` — OR-Set, add-wins            | ✅                                       |
| `Text` — Peritext-style rich text      | ✅ valued + boolean marks, expand rules  |
| Quill / Yjs Delta JSON interop         | ✅ `to_delta` / `from_delta`             |
| Yjs `lib0` + `StateVector`             | ✅ byte-identical primitives             |
| Yjs `Y.Update v1` snapshot encoder     | ✅ minimal — single-client `Y.Text`      |
| Anchor-based cursors / selections      | ✅                                       |
| Undo / redo (inverse-op API)           | ✅                                       |
| Tombstone GC + log compaction          | ✅                                       |
| Grapheme cluster handling (UAX #29)    | ✅                                       |
| Persistence (`Storage` trait, file)    | ✅ append-only log + atomic snapshots    |
| WASM bindings                          | ✅ wasm-bindgen                          |
| Python bindings                        | ✅ PyO3 abi3-py38                        |
| stateright model checker               | ✅ exhaustive interleaving search        |
| cargo-fuzz harness                     | ✅ 4 targets, 3.3M+ runs verified        |

The headline crate is on [crates.io][crate], source at [GitHub][gh],
docs at [docs.rs][docs]. Apache-2.0 / MIT dual-licensed.

[crate]: https://crates.io/crates/abyo-crdt
[gh]: https://github.com/abyo-software/abyo-crdt
[docs]: https://docs.rs/abyo-crdt

## Why a new CRDT library

The Rust CRDT landscape today:

- **Yjs / yrs** — battle-tested, large ecosystem, ~10 KLoC of
  optimized code. JS-first; the Rust API is a port of the JS one,
  and the binary wire format is `lib0` (tightly bound to JS
  conventions). For a Rust-native consumer, you're paying the
  conceptual overhead of a JS data model.
- **Automerge** — strong correctness story, Rust core. The JS API
  is the priority; perf has historically trailed Yjs. Best when you
  need byte-level history queries, less ideal as a high-throughput
  text engine.
- **Loro** — newer, Rust core, clever Eg-walker-inspired algorithms.
  JS bindings are the headline use case; the Rust API is functional
  but secondary.
- **diamond-types** — Joseph Gentle's research codebase, the
  reference implementation for Eg-walker. Explicitly a prototype.

If you're building a Tauri app, a Bevy game, a Rust backend that
needs to expose CRDT state to clients, or anything else where Rust is
the host language, none of these is built for you. Either you accept
the WASM-or-JS abstraction tax, or you build on a research prototype
that doesn't promise stability.

abyo-crdt is what I wanted: a Pure Rust library that takes the most
recent algorithms — Eg-walker for the event-log model, Fugue-Maximal
for non-interleaving list semantics, Peritext for rich text — and
exposes them through native Rust APIs with first-class
`#[derive(Serialize, Deserialize)]`, no `unsafe`, no JS interop in the
mental model.

## The list CRDT: Fugue-Maximal over an Eg-walker event log

The list CRDT is the heart of the library; everything else (text,
cursors, persistence) is built on top.

### Fugue-Maximal in one paragraph

Each item in the list is a node in an ordered tree:

```rust
struct Item<T> {
    id: OpId,                // Lamport-style (counter, replica)
    parent: Option<OpId>,    // None = top-level item
    side: Side,              // Left or Right of parent
    value: T,
    // tombstones, children, doc-order linked list...
}
```

Children of a node split into **left children** (visited before the
node in in-order traversal) and **right children** (visited after).
Within a side, siblings are sorted by `OpId` ascending — a Lamport-
based total order so older concurrent inserts are visited first.

When inserting at visible position `pos`:

1. If `pos == 0` and the document is non-empty: parent is the current
   first visible item, side = `Left`.
2. Else if `visible[pos-1]` has any right child (visible or
   tombstoned) **and** `pos < len`: parent is `visible[pos]`, side =
   `Left`.
3. Else: parent is `visible[pos-1]`, side = `Right`.

Why this rule? Because it guarantees the **non-interleaving property**:
contiguous bursts of inserts at the same position by different
replicas remain contiguous after merge. If Alice types "Hello" between
'a' and 'b' while Bob concurrently types "World" at the same spot,
the merged result is `aHelloWorldb` or `aWorldHellob` — never some
interleaved `aHWeolrllod b`. This is a meaningful upgrade over YATA
(Yjs's algorithm), which admits interleaving in pathological
concurrent edits.

The Fugue paper proves this rigorously. We verify it with hand-tuned
property tests in `tests/convergence.rs` and exhaustive interleaving
search in `tests/stateright_model.rs` (more on stateright below).

### The Eg-walker event log

Underneath the tree, every operation is recorded in `log: Vec<ListOp<T>>`
in the order it was observed. The visible sequence is recomputed by
walking the tree (or, for hot paths, by reading from the AVL OST
described next). This is the [Eg-walker model][eg-walker]: the
operation log is the canonical representation, and any local state
(visible-position index, format spans for rich text, undo stack) is
a derived view that can be rebuilt by replaying the log.

In practice this means:

- Persistence is "write the log to disk." We do snapshots + incremental
  log compaction in `src/storage.rs`, but the log is always the
  ground truth.
- Sync between replicas is "send each other ops." `ops_since(version)`
  gives you everything a peer hasn't seen.
- Replication is causal — the [`OpId`] is a Lamport timestamp
  `(counter, replica)`, and we maintain the invariant that every op's
  parent has a smaller `OpId`, so sorting the merge candidates by
  `OpId` ASC and applying in order automatically respects causality.

[`OpId`]: https://docs.rs/abyo-crdt/latest/abyo_crdt/struct.OpId.html

## The order-statistic AVL tree (and the bug it caught)

Naive Fugue-Maximal computes the visible sequence by walking the tree
on every read. That's `O(N)` per `get(pos)`, `iter()`, even
`determine_anchor()` during local insert. For a 1,000-char document
that's 22 ms per insert in the v0.1 implementation — `O(N²)` total to
build the document.

The fix is an **augmented order-statistic tree**: a balanced binary
search tree where the "key" is the in-order position rather than a
value comparison, and each node carries two subtree counts:

```rust
struct Node<T> {
    key: T,                     // OpId in our case
    visible: bool,              // tombstones flip this without restructuring
    parent: NodeId,
    left: NodeId,
    right: NodeId,
    height: i8,                 // for AVL balance
    visible_count: u32,         // # of visible items in this subtree
    total_count: u32,           // # of all items in this subtree
}
```

The two counts give us:

- `at_visible(rank)` — find the `i`-th visible node by descending
  the tree, comparing rank to subtree counts. `O(log N)`.
- `at_total(rank)` — same but counting tombstones too.
- `visible_position_of(key)` and `total_position_of(key)` — find a
  node by key (via a `HashMap<Key, NodeId>` companion), then walk
  parent pointers up to root accumulating left-subtree counts.
  `O(log N)`.
- `set_visible(key, bool)` — flip the bit, walk parent chain
  recomputing `visible_count`. No restructuring. `O(log N)`.

Tombstones flip the `visible` flag without restructuring the tree, so
deletes are also `O(log N)` (in the v0.1 implementation each delete
recomputed the visible sequence).

The implementation is in [`src/ost.rs`][ost], ~860 lines including
tests. Slab-based — nodes live in a `Vec<Option<Node>>` indexed by
stable `NodeId`s, no `unsafe`, no raw pointers. The free list
recycles slots when nodes are GC'd.

[ost]: https://github.com/abyo-software/abyo-crdt/blob/main/src/ost.rs

### The doc-order linked list

The OST gives us position queries, but we also need to know *where*
in document order a freshly-inserted item lands so we can splice it
into the OST at the right total rank. For a remote insert with parent
`P` and side `S`, the item's position depends on the CRDT tree
structure — and finding it by walking up `P`'s parent chain is `O(N)`
worst case for a left-chain document.

The fix: every `Item` carries `prev_doc` and `next_doc` pointers — a
maintained doubly-linked list in document order. On insert we read the
neighbors of the affected siblings to compute the new item's
predecessor in `O(1)` for the common cases (prepend, append, only-
sibling) and `O(depth-of-relevant-subtree)` otherwise.

Concretely: prepending 1,000 characters at position 0 went from
`O(N²)` (22 ms in v0.1, then 18 ms in a broken intermediate where I
forgot the prev/next maintenance) to `O(N log N)` (502 µs). Append
1,000 characters dropped from 22 ms to 557 µs. Random middle inserts
land in the same regime.

### The bug adversarial testing caught

The AVL implementation passed property-based testing — random insert
sequences with `check_invariants()` at every step, 1,000+ inserts
each, multiple seeds. It didn't catch this:

```rust
#[test]
fn adversarial_inserts_then_remove_root_repeatedly() {
    let mut t = OrderTree::<u32>::new();
    for i in 0..200u32 {
        t.insert_at_total(t.total_len(), i, true);
    }
    // Repeatedly remove the median.
    for _ in 0..150 {
        let mid = t.total_len() / 2;
        let val = *t.at_total(mid).unwrap();
        t.remove(&val);
        t.check_invariants();   // <-- failed
    }
}
```

The check-invariants assertion failed on iteration 50-something:
*node 101 has wrong parent: stored `NIL`, expected `103`*. Standard
BST deletion in the two-children case promotes the in-order successor
(`succ`) up to take the deleted node's slot:

```text
deleting `id` with two children
  ├ left
  └ right ──→ ... ──→ leftmost = succ
```

In my first cut, after detaching `succ` from its original position
and giving it `id`'s left and right subtrees, I set `succ.parent = NIL`
and *then* called `replace_in_parent(id, succ)`. The latter updated
the original parent's child pointer to `succ`, but `succ.parent` was
already `NIL` and never got updated to point at the original parent.
Net effect: `succ` is now the child of someone, but nobody knows it.
The next time we tried to walk up from a descendant of `succ` to the
root, we hit `NIL` halfway and the structural invariant exploded.

The diff was one line:

```diff
-self.node_mut(succ).parent = NIL;
+self.node_mut(succ).parent = parent;     // id's original parent
```

Property testing missed it because removing the median repeatedly is
an unusual pattern — most random-input generators distribute removals
across the tree, and a parent-pointer corruption deeper in the tree
gets papered over by subsequent operations that rewrite those
pointers. The adversarial test forces the path through the
two-children-direct-child branch over and over until the corruption
accumulates somewhere visible.

I would not have caught this bug without writing the specific
adversarial test. It was caught two days before the v0.4 launch. This
is the kind of thing that motivates the "more than property testing"
part of the verification story.

## The Peritext rich-text layer

`Text` is a `List<char>` with format spans grafted on top, following
the Peritext model. Each format span is anchored to specific character
`OpId`s — not absolute indices — so the span tracks correctly across
concurrent inserts and deletes:

```rust
pub enum Anchor {
    Start,                              // beginning of doc
    End,                                // end of doc
    Char(OpId, AnchorSide),             // before or after a specific char
}

pub struct Span {
    id: OpId,
    start: Anchor,
    end: Anchor,
    name: String,                       // "bold", "italic", "href", ...
    value: SpanValue,                   // On | Off | Set(String) | Unset
}
```

**Boolean marks** (`bold`, `italic`) use `SpanValue::On` and
`SpanValue::Off`. **Valued marks** (`href`, `color`) use `Set(s)` and
`Unset`. The render logic walks every span in `OpId` order; later ops
override earlier ones for the same name, so concurrent set/set or
set/clear on overlapping ranges resolves deterministically via the
Lamport tiebreaker.

When a character is inserted in a span's middle, it inherits the
mark — that's just the in-order traversal doing its job. When a
character is inserted at a span boundary, what happens depends on
**stickiness**, which Peritext models as anchor-side: an `After(c)`
end anchor means the span ends *right after* `c`, so a new char
inserted *also right after* `c` is on the boundary side that doesn't
extend the span. To extend it (the typical "expand right" behavior
for bold), use `Before(next_char)` instead. We expose this via
`ExpandRule { None, Right, Left, Both }`:

```rust
text.set_mark(0..5, "bold", true);
// Default ExpandRule::None — typing at boundaries doesn't inherit.

text.set_mark_with_rule(0..5, "bold", SpanValue::On, ExpandRule::Right);
// "expand right": typing at the right boundary continues the bold.
```

For grapheme cluster awareness — the `👨‍👩‍👧` problem where one
user-perceived character is five Unicode scalars — we expose
`grapheme_count`, `grapheme_to_char_pos`, `char_to_grapheme_pos`,
`insert_grapheme_str`, and `delete_grapheme`. Backed by the
`unicode-segmentation` crate (UAX #29). The underlying CRDT still
operates on `char`s (any Unicode scalar value is independently
addressable), so emoji can in principle be split by adversarial
concurrent edits — the grapheme API is the right user-facing
interface, but it's a layer over the char-level CRDT, not a
modification of the underlying semantics. Documented as a known
limit; full grapheme-level CRDT semantics is out of scope for v0.4.

## Yjs / Quill interop, in four progressively-tighter layers

The most-asked-for question for any new CRDT library is: "can it talk
to Yjs?" The honest answer is: full Yjs binary protocol implementation
is several weeks of careful work that hasn't been done. What
abyo-crdt v0.4 does ship is four interop layers, each useful at a
different scope:

### 1. Quill / Yjs Delta JSON

`Text::to_delta()` and `Text::from_delta()` round-trip with the Quill
Delta format that Yjs (`Y.Text.toDelta()`), Quill, Slate, and
ProseMirror all use. Lossy in both directions — Delta is a snapshot,
not the full op log — but enough for the common "render rich text in
a Yjs-backed editor" case.

```rust
let mut text = Text::new(1);
text.insert_str(0, "Hello world");
text.set_mark(0..5, "bold", true);
text.set_value_mark(6..11, "color", Some("#ff0000"));

let delta = text.to_delta();
// → [{insert: "Hello", attributes: {bold: true}},
//    {insert: " "},
//    {insert: "world", attributes: {color: "#ff0000"}}]
```

### 2. `lib0` binary primitives

Yjs's serialization format (`lib0`) is a custom variable-length
encoding. We ship byte-identical implementations of the four
primitives the rest of Yjs is built on:
`write_var_uint` / `read_var_uint`, `write_var_int` / `read_var_int`,
`write_var_string` / `read_var_string`. Tests verify the canonical
examples (`127 → [0x7F]`, `128 → [0x80, 0x01]`, etc.) match what
JS Yjs produces.

### 3. State-vector exchange

`yjs_compat::StateVector::encode()` produces the byte-exact
representation of `Y.encodeStateVector(doc)`. Two replicas — one
Yjs in a browser, one abyo-crdt on a server — can negotiate which
ops to exchange in this matching wire format.

### 4. `Y.Update v1` snapshot encoder

`yjs_compat::snapshot_text_to_yjs_update(text)` produces a Y.Update v1
binary that `Y.Doc.applyUpdateV1` accepts, materializing the abyo
`Text` content as a `Y.Text` at root key `"abyo"`. **One-way snapshot
only**: format marks aren't transferred (use Delta for those), and
the Yjs IDs don't match the source `OpId`s — Yjs sees a
single-author chain. Covers the "Rust server bootstraps a Yjs
browser" handoff.

What's *not* in the box: full Y.Update bidirectional sync (the struct-
store has 11+ content types, deletion sets, GC entries, and several
type-specific encoding variants for `Y.Map`, `Y.Array`, `Y.XmlElement`,
etc. — replicating it byte-for-byte is several weeks of dedicated
work). For now, abyo-crdt clients should bootstrap from the Y.Update
snapshot and exchange ongoing changes via Delta JSON over a custom
transport.

## Cursors, undo, GC, persistence

The boring-but-essential ergonomics the v0.1 plan deferred:

### Cursors and selections

```rust
let cur = Cursor::at(&list, 5);     // anchored before char at pos 5
list.insert(0, 'X');                 // doc shifts right
assert_eq!(cur.resolve(&list), 6);   // cursor follows
```

`Cursor` is `Copy`, 16 bytes, serde-roundtrippable. Three variants:
`Start`, `End`, `Anchored { char_id, side }`. Anchors track concurrent
edits correctly — if other replicas insert text before the cursor, it
shifts; if they insert text after, it stays put. If the anchored
character is deleted, the cursor "falls to" the position the deleted
character occupied (via the OST's phantom-position lookup).

`Selection` is just two `Cursor`s and a `resolve(&list) -> Range`.

### Undo / redo

```rust
let op = list.insert(0, 'X');
let inverse = list.apply_inverse(&op).unwrap();
// op was an Insert; inverse is the Delete that tombstones it.
```

The caller manages the undo stack; abyo-crdt provides the inverse-op
primitive. For Insert, the inverse is a Delete tombstoning the
inserted item. For Delete, the inverse is a fresh Insert anchored to
the tombstone's left side, conjuring a new item that occupies the
deleted item's visible position. (This isn't position-preserving in
all concurrent scenarios — under heavy concurrent edits, restoring a
deleted character lands somewhere reasonable but not necessarily
*where the user expected*. CRDT undo is intrinsically best-effort.)

### Tombstone GC

```rust
let frontier = list.version().clone();    // or intersection of peer VVs
let removed = list.gc(&frontier);          // O(N), removes leaf tombstones
list.compact_log(&frontier);               // drops covered ops
```

Tombstoned items with no children are eligible for GC if all of their
ops (insert + deletes) are observed by every replica in `frontier`.
The current implementation only handles leaf tombstones — internal
tombstones that have descendants would require subtree reparenting,
which gets us into "what does the CRDT semantics mean here"
territory. Documented as v0.5 work.

### Persistence

```rust
let mut store = FileStorage::open("doc.crdt")?;

// Restore.
let mut list: List<char> = store.load_snapshot()
    .unwrap_or_else(|_| List::new(replica_id));
for op in store.load_ops_after_snapshot()? {
    list.apply(op)?;
}

// Edit. Persist incrementally.
let op = list.insert(list.len(), 'X');
store.append_op(&op)?;

// Periodic snapshot to compact the log.
store.snapshot(&list)?;
```

Each persisted file is a sequence of length-prefixed records: kind
byte (snapshot or op) + payload length + payload (bincode-encoded).
`append_op` is `flush()` + `sync_data()`'d for durability. `snapshot`
writes a tempfile and renames over the original — atomic, and acts as
log compaction (the old log records are gone). The trait
([`Storage`][storage]) has a `MemoryStorage` companion for tests, and
implementations targeting other storage backends (RocksDB, S3, etc.)
are 50-line wrappers.

[storage]: https://docs.rs/abyo-crdt/latest/abyo_crdt/storage/trait.Storage.html

## The verification story

abyo-crdt is alpha software. It's also, by every metric I can think
of, more verified than most Rust crates ship as 1.0:

### 167 unit / property / serde tests

```text
test result: 99 lib unit (including 16 OST + 13 yjs_compat
              + 8 cursor + 27 text + 11 list + 8 map + 7 counter
              + 8 set + 3 storage)
            30 list API
             7 list property
             5 list serde
             7 stress (3 ignored long-runners)
             6 v0.2 prop
             5 v0.2 serde
             3 text prop
             3 text serde
             2 stateright
```

### 70K+ randomized property cases

`tests/convergence.rs` generates random op sequences across 2/3/5
replicas and verifies all converge regardless of merge order. Default
proptest cadence is 256 cases per property × 7 properties = ~1.8 K
cases per CI run; under `PROPTEST_CASES=10000` it's 70 K cases.

### Exhaustive interleaving search via stateright

`tests/stateright_model.rs` uses the [stateright][stateright] model
checker to do **BFS over every possible interleaving** of a small
bounded operation set. Where proptest is randomized sampling, this is
exhaustive enumeration — if it passes, no input in the bounded
parameter space breaks convergence. Currently checked: 2-replica
model with 3 ops, 3-replica model with 1 op each.

[stateright]: https://crates.io/crates/stateright

### 3.3M+ fuzz runs across 4 targets

Four cargo-fuzz targets in [`fuzz/`][fuzz]:

- `list_apply` — arbitrary `ListOp<u8>` sequences applied to a fresh
  List, asserting no panic + idempotency. **657 K runs in 56 s.**
- `list_convergence` — two replicas driven with arbitrary
  insert/delete actions, then merged, asserting `to_vec()` agrees.
  **307 K runs in 61 s.**
- `text_delta` — random Quill Delta JSON in →
  `Text::from_delta` → `to_delta` round-trip. **195 K runs in 61 s.**
- `yjs_state_vector` — arbitrary bytes to `StateVector::decode`,
  asserting either a typed error or a value that re-encodes
  identically. **2.12 M runs in 61 s.**

Total: **3.28 M runs across the four targets in roughly four minutes
of in-session fuzz time, zero panics.** This is far less than a
production fuzz farm (the FerroSearch baseline is 24-hour fuzz
sessions on dedicated hardware), but it's an order of magnitude more
than most Rust crates run before publishing.

[fuzz]: https://github.com/abyo-software/abyo-crdt/tree/main/fuzz

### `cargo clippy --all-targets --all-features -D warnings` clean

Treating clippy as errors in CI catches a class of bugs (sign-cast
issues, redundant clones, unnecessary allocations) before they ship.
The crate is also `#![forbid(unsafe_code)]` — there is no `unsafe`
in any of the 8 K lines of library code.

### Adversarial unit tests for the OST

In addition to property testing, `src/ost.rs` ships hand-tuned
adversarial unit tests:
`adversarial_alternating_inserts`,
`adversarial_zigzag_inserts`,
`adversarial_inserts_then_remove_root_repeatedly` (the one that
caught the AVL bug),
`adversarial_set_visible_thrashing`. These are designed to maximize
rotation chains and contention on specific code paths that random
testing rarely exercises.

## Memory

`PeakAlloc`-instrumented bench, `benches/memory.rs`:

| scenario                                | peak heap |
| --------------------------------------- | --------: |
| `List<char>` append 1,000 chars         | 742 KB    |
| `List<char>` append 10,000 chars        | 5.93 MB   |
| `List<char>` 1,000-then-delete-half     | 742 KB    |
| `Text` plain 1,000 chars                | 972 KB    |
| `Text` 1,000 chars + 100 marks          | 972 KB    |

That's roughly 600 bytes per character — heavier than `yrs` would be
on the same workload. The bulk is HashMap overhead (the
`HashMap<OpId, Item<T>>` items map plus the OST's
`HashMap<OpId, NodeId>` reverse index) plus the
prev_doc / next_doc / left_children / right_children fields per
Item. Item layout optimization (packing the children Vecs into a
sentinel-terminated linked list, replacing the HashMap with a slab,
RLE-encoding contiguous-run inserts) is a v0.5 task with a clear
upper bound — Loro's `Item` is around 80 bytes, ours could realistically
land at 120-150 bytes per item with focused work.

For now: if you're storing a 100,000-character document, expect
~60 MB resident. That's within tolerance for desktop apps and most
backend contexts; for embedded or very-large-doc use cases, wait for
v0.5 or pre-trim with `gc()` aggressively.

## FFI bindings

The library is Rust-native, but `bindings/wasm` and `bindings/py`
expose `Text` to JavaScript and Python.

### WASM

```javascript
import init, { TextDoc } from "./pkg/abyo_crdt_wasm.js";
await init();

const doc = TextDoc.new(1n);          // BigInt replica id
doc.insert(0, "Hello, world!");
doc.setMark(0, 5, "bold", true);

const delta = doc.toDelta();          // Quill / Yjs Delta JSON
const yjsBin = doc.toYjsUpdateV1();   // Y.applyUpdateV1-compatible bytes

const bytes = doc.toBincode();        // Persist
const restored = TextDoc.fromBincode(bytes);
```

Built via `wasm-pack build --target web` in `bindings/wasm`.
`TextDoc` is the headline class; `U32List` shows the pattern for
custom `List<T>` adapters.

### Python

```python
from abyo_crdt_py import TextDoc

doc = TextDoc(replica_id=1)             # or TextDoc() for OS-random id
doc.insert(0, "Hello, world!")
doc.set_mark(0, 5, "bold", on=True)

print(doc.to_string())
yjs_bin = doc.to_yjs_update_v1()        # Y.Doc.applyUpdate-compatible bytes
state = doc.to_bincode()                # for pickle
```

Built via `maturin build --release` in `bindings/py`. PyO3 with
`abi3-py38` so a single wheel works on Python 3.8+.

Both bindings are independent Cargo workspaces — they don't pollute
the main `cargo build` with WebAssembly or Python toolchains.

## What's next

The honest scope assessment: abyo-crdt v0.4 is **feature-complete for
its stated targets, production-grade in engineering practice,
**alpha** in real-world adoption**. The gaps that keep it from being
labeled v1.0:

- **Real-world users.** The library has zero production deployments
  as of v0.4.0-alpha.1 release. The kinds of bugs that only surface
  under millions of ops across thousands of replicas haven't had a
  chance to surface. CI-grade fuzz ran for 4 minutes; a production
  fuzz farm runs 24 hours per night.
- **Full `Y.Update v1` bidirectional sync.** The snapshot encoder
  ships; the full struct-store with all 11+ content types and the
  deletion-set encoding is queued for v0.5.
- **Internal-tombstone GC.** Leaf tombstones are reclaimed; tombstones
  with descendants would require subtree reparenting and is queued
  for v0.5.
- **Item-layout memory optimization.** ~600 B/char is fine for
  desktop apps; getting to ~150 B/char (Loro-class) takes focused
  layout work.

Specific things queued for the v0.5 cycle, roughly in priority order:

1. Item layout optimization → 4× memory reduction target.
2. Internal-tombstone GC with subtree reparenting.
3. Full Y.Update v1 bidirectional sync.
4. Loro / Automerge comparison benchmarks (we only compared to yrs
   for v0.4).
5. A "production hardening" CI job: 24-hour fuzz, sanitizer runs,
   memory-leak detection.
6. TLA+ specification of the Fugue-Maximal merge semantics.

abyo-crdt is the second public release in the [`abyo-software`][abyo]
paper-to-crate series, sibling to [`exaloglog`][exaloglog]. The
queued backlog continues with RaBitQ for vector quantization, Ribbon
+ InfiniFilter for approximate-membership filters, ChalametPIR for
private information retrieval, and BMP + Seismic for learned-sparse
retrieval. abyo-crdt itself is independent of the Ferro product line
— Tauri and Bevy and the broader local-first community are the
primary consumers — but it's part of the same engineering posture:
take a recent paper, ship a Pure Rust implementation, hold the work
to a verification bar above what the paper itself proves, and put the
numbers up front.

If you're already reaching for Yjs or Automerge in a Rust context,
abyo-crdt is worth a look — especially if you're append-heavy,
storage-constrained, or need something that doesn't apologize for the
host language. Source on [GitHub][gh]. Crate on [crates.io][crate].
Docs on [docs.rs][docs]. Bug reports backed by failing tests, perf
numbers from your workloads, and PRs welcome.

[abyo]: https://github.com/abyo-software
[exaloglog]: https://crates.io/crates/exaloglog
