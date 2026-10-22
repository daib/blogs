# Snapshotting String Interning

## Introduction

A common technique for managing large collections of strings is string
interning: each distinct string is stored once and replaced everywhere
else with a small integer id. This gives you cheap deduplication (a
string that appears a million times is stored once) and cheap comparison
(comparing two ids is comparing two integers, not two strings).

Interning is usually backed by two structures: `id_to_string`, which
resolves an id back to its string, and `string_to_id`, which checks
whether a string has already been interned and returns its existing id if
so. Both work well in memory. The question this note is about is what
happens when you want to write that state to disk and load it back later
— ideally without paying the cost of re-interning every string from
scratch.

Rebuilding from scratch means re-hashing every string, re-inserting it
into a hash table, and re-allocating storage for it — real CPU work and
real latency, proportional to how many strings you have, paid again every
single time you load. For a table with hundreds of thousands of entries,
that cost dominates load time far more than reading the bytes off disk
ever does. The goal is to make loading closer to "read bytes, done" —
no re-hashing, no re-insertion, no per-string work at all.

## Why a hash table resists this by default

Serializing a hash table naively — dumping its key/value pairs into a flat
array — throws away the one thing that made it fast: the table's internal
bucket layout, computed from each key's hash. Reload that flat array and
you're back to re-hashing and re-inserting every entry to get a working
table again, which defeats the entire point.

The fix isn't a cleverer serialization format. It's restructuring the two
tables so that one of them never needed hashing to begin with, and the
other can be restored as a flat block with no reconstruction at all.

## `id_to_string`: make the id the location

Store every interned string, once, in a flat buffer — one contiguous
region of memory (or several fixed-size regions, chained in order, if you
want to avoid ever reallocating or moving existing entries as new strings
arrive).

An id is not a plain array index, because strings vary in length and a
plain index can't locate a variable-length entry without either a
separate offset table or padding every string to a fixed size — both real
costs. Instead, an id encodes **a location**: which region the string
lives in, plus a byte offset within that region. Resolving a string from
its id is then pure arithmetic — split the id into its region and offset,
add the offset to that region's base address, read the string out. No
table, no indirection, nothing that could have moved.

Because the id already *is* the string's address, this whole region can be
written to disk verbatim and mapped back into memory on load, at
whatever address the operating system happens to choose, with zero
restoration cost — the bytes already mean exactly the same thing they did
before, since nothing inside them depends on where they happen to sit in
memory this time.

One further optimization worth making explicit: compute each string's hash
once, at the moment it's first interned, and store that hash alongside the
string itself in the same region. This is the detail that makes the
second table cheap, below.

## `string_to_id`: a hash table with nothing left to rebuild

You still need the reverse lookup — given a string, has it already been
interned, and what's its id — and that genuinely does need a hash table.
The trick is building one whose entire layout is plain, flat data with no
pointers and no library-specific internal structure, so the table itself
can be written and restored as one contiguous block.

Each slot in the table holds only: the string's hash and its id — nothing
else. The string's actual content never needs to live inside this table
at all, because confirming a real match (rather than a hash collision) can
be done by looking up the candidate id in `id_to_string` and comparing
against the string stored there. That's the payoff of interning the hash
alongside the string in the first structure: a lookup never needs to
re-hash anything, on either side of the comparison, ever.

Because every slot in this table is now a small, fixed-size, pointer-free
struct, the entire table is just an array of uniform records. Writing it
to disk is a direct copy of that array. Loading it back is one block copy
into memory, with no re-hashing, no re-insertion, no per-entry work of
any kind.

## What this buys you

Put together, loading an interned-string snapshot from disk becomes:

1. Map or copy the `id_to_string` region(s) into memory — raw bytes, no
   transformation.
2. Copy the `string_to_id` table's slot array into memory — raw bytes, no
   transformation.

Every id resolves correctly immediately, because ids are locations, not
references into a structure that has to be rebuilt first. Every reverse
lookup works immediately, because the hash table's layout was never lost
in the first place — it was never allowed to depend on anything that
serialization would throw away.

The general principle, stripped of the specifics above: a data structure
is cheap to snapshot exactly when every reference inside it is a
self-describing location rather than a pointer or an index into something
else, and every container it uses has a layout you control completely,
rather than one supplied by a library that reserves the right to change
it. Once both are true, the structure's in-memory form and its on-disk
form are the same thing, and "loading" stops being a meaningful verb —
there's nothing left to do but move bytes.

This is one instance of a broader family of techniques worth calling
**Active Data Structures** — structures designed so the bytes you'd write
to disk are already the structure you'd query, rather than a description
of it that needs interpreting first. The same idea shows up again, at a
different scale, in how a relocatable graph can encode node addresses; that's
a separate piece on its own.

## A real limitation: addressable size is fixed upfront, not discovered

Encoding a location inside a fixed-width id has a real cost, worth stating
plainly rather than glossing over: the total amount of data you can
address is bounded by however many bits you're willing to spend on the
id, and that choice has to be made before you know how much data you'll
actually have.

A 32-bit id gives you at most 4 billion distinct byte-offsets to work
with, and in practice less than that — the bits split between "how many
regions" and "how large each region can be," so a design favoring many
regions sacrifices how large any one region can grow, and vice versa.
Picking that split is a real, upfront tradeoff, not a detail you can defer:
get it wrong and you either run out of address space as your dataset
grows, or waste bits reserving headroom you never use.

This isn't a flaw to apologize for — it's the actual price of the
technique, the same way giving up arbitrary pointers is. Zero-cost loading
in exchange for a hard, chosen ceiling on addressable size is a trade worth
making deliberately, with real numbers behind the choice, not a limitation
that quietly shows up later.

Worth keeping the ceiling in perspective, though: 4 billion bytes is a
real constraint, not a crippling one. Most individual streaming blobs or
snapshot chunks — a bounded slice of recent data, not an entire dataset's
history — land well inside that range, so a 32-bit id is often simply
enough, not a compromise you feel day to day. The limitation matters when
a single structure is expected to grow without bound; it's close to a
non-issue when the structure is naturally chunked, each chunk snapshotted
on its own.
