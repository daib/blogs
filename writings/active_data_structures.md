# Active Data Structures

There's an old idea from parallel computing called Active Messages: instead
of a message arriving as inert data that something downstream has to parse
and act on, the message carries a handler, and execution starts the
instant it lands. No buffering step. No interpretation pass standing
between "arrived" and "running."

Most on-disk formats never got that memo. The usual deal is: write your
data in some format optimized for size or portability, and pay an
interpretation cost on every single load, forever — parse it, remap its
IDs, rebuild its indexes, re-allocate its objects, and only then do you
have something you can actually query. That's a reasonable trade when the
format needs to be compact, or readable by other tools, or stable across
years of schema changes. It's a bad trade when the only thing standing
between "bytes on disk" and "a usable structure" is work you already did
once, at write time, and are now redoing identically, from scratch, every
time you load.

Stated as two flows, the difference is exactly one missing step:

```
Normal:  serialize -> store -> load -> deserialize -> use
Faster:  serialize -> store -> load ------------------> use
```

Removing "deserialize" isn't a smaller, cheaper version of it. It's the
whole step gone — nothing to parse, remap, or rebuild between "the bytes
are in memory" and "the structure is ready to query." Every one of the
techniques below is really just an answer to one question: what has to be
true at *serialize* time so that *deserialize* is never needed at all?

Worth stating the ceiling this aims at, since it's a real one, not just a
rhetorical flourish: if the in-memory structure you're storing is already
the optimal structure for the queries you run against it, and the bytes
you've stored are the minimal bytes needed to represent it, then nothing
can beat the Faster flow — there is no faster way to go from disk to a
queryable structure than *not transforming it at all*. Every other
approach is, at best, catching up to a floor this flow already sits on.

An **Active Data Structure** is the data equivalent of an Active Message:
instead of serializing a description of a structure that needs
interpreting, you write out the structure itself — the actual bytes a
query runs against — and loading becomes "map the bytes in, done." No
parse step. No reconstruction. The structure doesn't arrive inert; it
arrives ready to run.

## The one rule that makes this possible

Everything below reduces to a single requirement: **nothing inside the
structure can depend on where it happens to sit in memory.**

A normal pointer fails this immediately — it's only meaningful in the
address space that created it. Swap the bytes to a different process, or
even reload them into the same process at a different address, and every
pointer inside is now garbage. The fix is to replace every pointer with an
**ID that encodes a location directly** — an offset, an index, a
coordinate — something whose meaning doesn't depend on where the OS
decided to put the bytes this time.

Once every reference in a structure works that way, the structure's
in-memory form and its on-disk form are the same thing. There's nothing
left to translate. "Loading" stops being a verb with real work behind it.

## Two worked examples, same rule, different data

### String interning

[Covered in detail elsewhere](snapshotting_string_interning.md), but the
short version: an interned string's id can encode exactly where the
string lives — which memory region, what byte offset — so resolving a
string from its id is pure arithmetic, no lookup table required. The
reverse lookup (string → id) still needs a real hash table, but it can be
built so that every slot is a flat, pointer-free struct — a hash and an id,
nothing else — which makes the whole table a single block of bytes you
can write and restore with one copy, no re-hashing.

### A relocatable graph

The same rule applies just as cleanly to something with a much richer
reference structure: a graph where every node needs to reach its
neighbors — a vector-search index built on a navigable proximity graph
is a direct example, with each node holding a handful of edges to its
nearest neighbors at several levels of a hierarchy.

Collocate each node's data with its own edges in one fixed-size arena
block, so that reading a node already gives you its neighbors in the same
cache line, and encode each node's id as a location inside that scheme —
split the available bits between *which block* a node lives in and *what
offset* inside that block:

```
NodeId (32 bits): [ block index: 14 bits | offset: 18 bits ]
```

With a fixed block size and reasonable alignment, those bits comfortably
address tens of gigabytes of graph — enough for the vast majority of
real workloads without needing anything wider. Resolving a neighbor is
the same arithmetic as before: split the id, index into the block array,
add the offset. Walking the graph is a sequence of these resolutions,
every edge traversal from one node to the next, each one meaningless in
exactly the way a pointer dereference is meaningless, except that this one
never breaks when the block holding it gets mapped in at a different
address than the one that originally wrote it.

The payoff shows up directly in cold-start behavior: a graph built this
way can be written to disk once and mapped back into memory later with
zero reconstruction — no re-linking edges, no rebuilding an index, no
per-node work of any kind. The bytes you read are already the graph.

This isn't hypothetical. [MemWeaver](https://github.com/daib/mem_weaver), an
open-source vector index built on exactly this scheme, was benchmarked
against LanceDB on SIFT1M (1M vectors, dim=128):

| | Cold time-to-first-answer | Warm p50 latency | Warm peak QPS |
|---|---|---|---|
| MemWeaver (eager, Active Data Structure) | ~130 ms | 0.25–0.34 ms | ~14,000–15,400 |
| LanceDB (lazy, mmap-backed) | ~212–264 ms | 1.4–5.2 ms | ~1,220–1,320 |

LanceDB's lazy-loading model only pages in whatever a given query touches,
which is a genuinely different and reasonable strategy — it should win
when an index is far larger than available memory and only a small
fraction of it is ever queried. But against this dataset, closing the
distance between MemWeaver's on-disk bytes and its runnable graph entirely
— rather than trying to make the *loading itself* faster — won on both
axes: faster to first answer cold, and 10–50x faster once warm, since
there's no per-query indirection left once the bytes are the graph.

## The shared shape underneath both

Strings and graph nodes look nothing alike on the surface, but the fix is
identical in both cases, worth stating as the actual, reusable principle
rather than two separate tricks that happen to resemble each other:

- Replace every pointer with a **self-describing location** — an id that
  tells you exactly where to find what it refers to, without needing a
  table or a process-specific address to resolve it.
- Own the layout of every container you build this on top of. A library
  structure with a private internal layout — however clever — can't be
  trusted to keep that layout stable across a version bump, let alone
  across being written to disk and read back by different code. If you
  didn't choose the byte layout, you can't snapshot it.

Everything that follows from those two constraints — zero-copy loading,
portability across processes and machines, cold-start costs that approach
the physical floor of moving bytes rather than the cost of re-deriving a
structure — is a direct, almost mechanical consequence, not a separate
set of optimizations you have to go looking for individually.

## Where it stops working, and why that's fine

Not everything qualifies, and it's worth being honest about the boundary
rather than overselling the technique.

Anything whose natural representation genuinely requires an unbounded,
dynamically-growing reference — an object that might need to point
anywhere in an address space that keeps expanding, with no natural
chunking boundary — doesn't fit neatly into a fixed-width, location-
encoded id. The fix there usually isn't to abandon the approach; it's to
find the natural chunk boundary your data already has (a time window, a
dataset partition, a single file's worth of graph) and apply the technique
within each chunk, accepting a real, upfront ceiling on how much any one
chunk can hold in exchange for every chunk loading for free. That ceiling
is a design parameter you choose deliberately, with real numbers behind
it — not a flaw that shows up as a surprise later.

The honest summary: Active Data Structures don't make every loading
problem disappear. They make the loading problem disappear for the large,
common class of structures that are read far more often than they're
restructured — which turns out to be most of what a lot of real systems
spend their cold-start time rebuilding, for no better reason than that
nobody had yet asked whether rebuilding it was ever necessary in the
first place.
