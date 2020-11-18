# Snapshots

## Reference counting

Check backreferences.txt for a more complete explanation of how each reference
type works.  For the sake of this document, we will refer to the standard
reference that gets propagated by the referring root as "normal" backreferences,
and "full backref" as the reference that is propagated using the bytenr as the
parent.  Consider the following tree

       +----------+
       | Root ABC |
       +-----+----+
             |
           +-+-+
           | A |
      +----+-+-+----+
      |      |      |
      |      |      |
    +-+-+  +-+-+  +-+-+
    | B |  | C |  | D |
    +-+-+  +-+-+  +-+-+
      |      |      |
    +-+-+  +-+-+  +-+-+
    | E |  | F |  | G |
    +---+  +---+  +---+

### Normal COW

Each of these blocks have a "normal" backref, all referring belonging to Root
ABC.  They were allocated by root 1, and thus they only have normal root refs.
Now consider the normal COW case, where we're COW'ing to block E.

           +----+
           | A' |
      +----+--+-+----+
      |       |      |
      |       |      |
    +-+--+  +-+-+  +-+-+
    | B' |  | C |  | D |
    +-+--+  +-+-+  +-+-+
      |       |      |
    +-+--+  +-+-+  +-+-+
    | E' |  | F |  | G |
    +----+  +---+  +---+

1. COW Block A
    1. Allocate block A', add a normal ref for ABC.
    2. Drop ref for ABC from A, free block as it's unreferenced.
2. COW Block B
    1. Allocate block B', add a normal ref for ABC.
    2. Drop ref for ABC from B, free block as it's unreferenced.
3. COW Block E
    1. Allocate block E', add a normal ref for ABC.
    2. Drop ref for ABC from E, free block as it's unreferenced.

Block A, B, and E have their "Normal" reference dropped for Root ABC, and block
A', B', and E' have a new entry added with a reference to Root ABC.  The
reference counts for C and D were untouched in this case.

### Snapshot

Now consider the original tree again, and this time we create a snapshot of Root
ABC.  Block A is copied in it's entirety, and references to all of the children
pointed at by A are updated with a reference from Root DEF. 

    +--------+              +--------+
    |Root ABC|              |Root DEF|
    +----+---+              +---+----+
         |                      |
         |                      |
       +-+-+                  +-+--+
       | A |                  | A' |
       +------------+--------++----+
           |        |        |
           |        |        |
         +---+    +-+-+    +-+-+
         | B |    | C |    | D |
         +-+-+    +-+-+    +-+-+
           |        |        |
         +-+-+    +-+-+    +-+-+
         | E |    | F |    | G |
         +---+    +---+    +---+

1. Allocate A'.
2. Copy A into A'.
3. Iterate all blocks, add a reference each child of A' and add a reference for
   root DEF.

The extent references at this point are as follows

| A  | 1 | Normal ABC, Normal DEF |
|----|---|------------------------|
| A' | 1 | Normal ABC, Normal DEF |
| B  | 2 | Normal ABC, Normal DEF |
| C  | 2 | Normal ABC, Normal DEF |
| D  | 2 | Normal ABC, Normal DEF |
| E  | 1 | Normal ABC             |
| F  | 1 | Normal ABC             |
| G  | 1 | Normal ABC             |

### COW E from ABC with a snapshot

Taking the above tree example with the snapshot DEF, we are going to COW down to
block E from root ABC.  This works similarly as the normal COW case, except we
will convert block B and block E into full backref references.

    +-----+
    | A'' +------------------+------------+
    +--+--+                  |            |
       |                     |            |      +----+
       |          +------------------------------+ A' |
       |          |          |            |      +----+
       |          |          |            |
       |          |          |            |
    +--+-+       ++--+     +-+-+        +-+-+
    | B' |       | B |     | C |        | D |
    +--+-+       ++--+     +-+-+        +-+-+
       |          |          |            |
       |          |          |            |
       |          |          |            |
       |          |          |            |
    +--+-+      +-+-+      +-+-+        +-+-+
    | E' |      | E |      | F |        | G |
    +----+      +---+      +---+        +---+

1. COW Block A'
    1. Allocate block A'', add a normal ref for ABC.
    2. Drop ref for ABC from A', free block as it's unreferenced.
2. COW Block B
    1. Allocate block B', add a normal ref for ABC.
    2. Check if B can be shared, which it can be, look up the reference count,
       which is 2.
    3. Because the ref count is > 1 and ABC is the owner of B, call
       `btrfs_inc_ref` on B with full_backref == 1, which walks all of the
       children of B and add's a ref pointing at the bytenr of B.
    4. Drop the ref for ABC from B.  B is not free'd as it still has 1 reference.
3. COW Block E
    1. Allocate block E', add a normal ref for ABC.
    2. Check if E can be shared, which it can be, look up the reference count,
       which is 2, one normal ref for ABC, and 1 full backref from B.
    3. Because the ref count is > 1 and ABC is the owner of E, call
       `btrfs_inc_ref` on E with full_backref == 1, which walks all of the
       children of E and add's a ref pointing at the bytenr of E where
       appropriate (file extents and such).
    4. Drop the ref for ABC from E.  E is not free'd as it still has 1 reference.

B and E are no longer referenced by root ABC.  When B was COW'ed to B', we
dropped our reference for root ABC, and we added the `FULL_BACKREF` flag to it's
extent reference, because it is no longer referenced by the owner root.  The
owner root is determined with `btrfs_header_owner()` on the `extent_buffer`.
This is set to the root who originally allocated the block.  At this point the
references are as follows

| Block | Ref Count | References             |
|-------|-----------|------------------------|
| A''   | 1         | Normal ABC             |
| A'    | 1         | Normal DEF             |
| B     | 1         | Normal DEF             |
| B'    | 1         | Normal ABC             |
| C     | 2         | Normal ABC, Normal DEF |
| D     | 2         | Normal ABC, Normal DEF |
| E     | 1         | Full backref B         |
| E'    | 1         | Normal ABC             |
| F     | 1         | Normal ABC             |
| G     | 1         | Normal ABC             |

### COW E from DEF with a snapshot

Continuing with the above tree state, we COW down to block E from root DEF

    +-----+
    | A'' +------------------+------------+
    +--+--+                  |            |
       |                     |            |      +------+
       |          +------------------------------+ A''' |
       |          |          |            |      +------+
       |          |          |            |
       |          |          |            |
    +--+-+      +-+-+      +-+-+        +-+-+
    | B' |      |B''|      | C |        | D |
    +--+-+      +-+-+      +-+-+        +-+-+
       |          |          |            |
       |          |          |            |
       |          |          |            |
       |          |          |            |
    +--+-+      +-+-+      +-+-+        +-+-+
    | E' |      |E''|      | F |        | G |
    +----+      +---+      +---+        +---+

1. COW Block A'.
    1. Allocate block A''', add a normal ref for DEF.
    2. Drop ref for DEF from A', free block as it's unreferenced.
2. COW Block B.
    1. Allocate block B'', add a normal ref for DEF.
    2. Check if the block can be shared, which it can, so look up the reference
       count and extent flags.
    3. We have `FULL_BACKREF` set, which means we must call `btrfs_inc_ref` with
       `full_backref == 0` to add a normal reference for DEF to all of the children
       of B.
    4. Because we have `FULL_BACKREF` and a reference count of 1, call
       `btrfs_dec_ref` with `full_backref == 1` on B to drop the reference from
       the bytenr of B from all the children of B.
    5. Drop our normal reference for DEF from B, the reference count is now 0 and
       it can be freed.
3. COW Block E.
    1. Allocate block E'', add a normal ref for DEF.
    2. Check if the block can be shared, which it can, so look up the reference
       count and extent flags.
    3. We have `FULL_BACKREF` set, which means we must call `btrfs_inc_ref` with
       `full_backref == 0` to add a normal reference for DEF to all of the children
       of E.
    4. Because we have `FULL_BACKREF` and a reference count of 1, call
       `btrfs_dec_ref` with `full_backref == 1` on E to drop the reference from
       the bytenr of E from all the children of E.
    5. Drop our normal reference for DEF from E, the reference count is now 0 and
       it can be freed.

At this point B and E are free'd, as the no longer have any root referencing
them.

| Block | Ref Count | References             |
|-------|-----------|------------------------|
| A''   | 1         | Normal ABC             |
| A'''  | 1         | Normal DEF             |
| B''   | 1         | Normal DEF             |
| B'    | 1         | Normal ABC             |
| C     | 2         | Normal ABC, Normal DEF |
| D     | 2         | Normal ABC, Normal DEF |
| E'    | 1         | Normal ABC             |
| E''   | 1         | Normal DEF             |
| F     | 1         | Normal ABC             |
| G     | 1         | Normal ABC             |

