# Relocation

## Determining the blocks to relocate

This is simple enough, we iterate through the relevant `btrfs_block_group`'s
that we are relocating for this balance, and we search for any bytenr's that
fall in each block groups range.  This is done by searching the `extent_tree`,
starting at `btrfs_block_group->start` through `btrfs_block_group->start +
btrfs_block_group->length`.  Each bytenr we hit we need to build a backref cache
for, and we need to determine the `node_key`, which is simply the key of the
first item in that node.  This is used for indirect reference resolution.

## Backref cache, the basics

The integral part to relocation is the backref cache.  The crux of this work is
accomplished in `build_backref_tree`.  The core structures for this are the
following, abbreviated to highlight the elements of the structs that important for
this section only.

```
struct btrfs_backref_node {
	/* The bytenr of the node/leaf that this object represents. */
	u64 bytenr;

	/* The list of edges that link us to our parent nodes. */
	struct list_head upper;

	/* The list of edges that link us to our children nodes. */
	struct list_head lower;
};

struct btrfs_backref_edge {
	/* The lists to link between two different nodes. */
	struct list_head list[2];

	/* The actual nodes that this edge connects. */
	struct btrfs_backref_node *node[2];
};
```

In order to illustrate how this fits together logically, we will use the
following example tree.

    +----------+    +----------------+
    | Root 256 |    | Reloc root 256 |
    +-----+----+    +--------+-------+
          |                  |
       +--+--+           +---+---+
       |  A  |           |   A'  |
       +--+--+           +-----+-+
          |                    |
          |      +-----+       |
          +------+  B  +-------+
                 +-+-+-+
                   | |
       +-----+     | |    +-----+
       |  C  +-----+ +----+  D  |
       +-----+            +-----+

If we were to build a backref tree for C, it would look something like this

          +-->Backref C
          |
     lower+
    EDGE1
     upper+
          |
          +-->Backref node B<--+
          |                    |
     lower+                    +lower
    EDGE2                        EDGE3
     upper+                    +upper
          |                    |
          +-->Backref node A   |
                               |
              Backref node A'<-+

And logically would look something like the following

    LOWER = 0
    UPPER = 1

    Backref node C
     ->lower = []
     ->upper = [EDGE1->list[LOWER]]

    EDGE1
     ->list[LOWER] = [C->upper]
     ->list[UPPER] = [B->lower]
     ->node[LOWER] = C
     ->node[UPPER] = B

    Backref node B
     ->lower = [EDGE1->list[UPPER]]
     ->upper = [EDGE2->list[LOWER], EDGE3->list[LOWER]]

    EDGE2
     ->list[LOWER] = [B->upper]
     ->list[UPPER] = [A->lower]
     ->node[LOWER] = B
     ->node[UPPER] = A

    EDGE3
     ->list[LOWER] = [B->upper]
     ->list[UPPER] = [A'->lower]
     ->node[LOWER] = B
     ->node[UPPER] = A'

    Backref node A
     ->lower = [EDGE2->list[UPPER]]
     ->upper = []

    Backref node A'
     ->lower = [EDGE3->list[UPPER]]
     ->upper = []

The code can be confusing, because you see things like

```
list_add(&node->upper, &edge->list[LOWER]);
```

but this is because the `btrfs_backref_edge` describes the path between two
`btrfs_backref_node`'s, and the nodes themselves point at lower and upper from
their point of view in the tree.

## Backref cache building specifics

When we do `btrfs_backref_add_tree_node()` we are attempting to build the entire
backref tree as described above.  This is done by looking up any references to
the given bytenr, and building the edges from there.  There are two cases here

### `BTRFS_SHARED_BLOCK_REF_KEY`

In this case the reference to our block is the bytenr for our parent.  This is a
straightforward case, we look up that bytenr in our cache and link it to the
existing node with our new edge if it exists in cache.  If it does not exist in
cache, we link the `edge->list[LOWER]` to our current and link
`edge->list[UPPER]` to our `cache->pending` list so we know we must go look up
the parents of that backref node.

### `BTRFS_TREE_BLOCK_REF_KEY`

This is what is described as an indirect reference, because at this point we
only know our current node level and the root that holds a reference for this
block.  From here we use our `btrfs_key` that we looked up as the first key in
our node from the relocation step to search down the root that owns our
reference.  We use `->lowest_level = current_level + 1` in order to find our
parent node.  Once we find this block we simply walk up the `btrfs_path`, adding
edges for each of the nodes in our path down to the block that we are
processing.  Any nodes that are not in cache have their corresponding
`edge->list[UPPER]` list linked into `cache->pending_list` to process next for
building the completed backref cache.

## `reloc_inode`

The `reloc_inode` is a special inode that is created each time we attempt to
relocate a block group.  During the `MOVE_DATA_EXTENTS` phase of relocation,
which is the first stage, any data extents we find are copied into the
`reloc_inode`.  The copying takes place in `relocate_file_extent_cluster`, which
does the following.

1. Preallocate the space in the inode for the data extents.  The offset in the
   inode is the offset into the block group for the data.  For example, if
   you are relocating a block group that starts at 1GIB and you are moving the
   range [1GIB, 1.5GIB) then the offset will be [0, 0.5GIB).

2. Change the extent mapping for the inode.  Because we need to read the
   original data from disk, we modify the extent mapping of the file so that the
   logical offsets of the inode map to the original bytenr on disk.  Taking the
   previous example, this would make the inode think that offset 0 mapped to
   bytenr 1GIB on disk, instead of the bytenr that we preallocated.  This is in
   preparation for the next step.

3. Read in the data and mark it dirty.  Because we have pinned our extent
   mapping into place `btrfs_get_extent` will map our read IO's to the original
   location on disk.  However when we preform the writeback of the inode we will
   look up the actual file extents on disk in order to write out the copied data
   to the new location.

This is the only thing that happens during the `MOVE_DATA_EXTENTS` with the
`reloc_inode`.  Once we've completed this phase we move onto the
`UPDATE_DATA_PTRS` phase.  This phase is where we update the file extent
pointers to point at the new data location.  This happens via
`btrfs_reloc_cow_block`, at this stage we look up any leafs that point at our
data and we relocate those leafs the same way we relocate any metadata.  This
triggers the `btrfs_reloc_cow_block` path which in turn triggers
`replace_file_extents`.  From here we look up the inode that owns the file
extent, drop the extent cache for that range of the inode, and set the new
bytenr location, adding a reference for the inode and dropping the reference to
the old bytenr that the file extent previously pointed at.

## The mechanics of relocating

There are a few mechanical steps to relocating a block group.

1. Mark the block group read only.  This is needed to make sure nobody tries to
   make allocations from this block group.
2. Create the `reloc_inode`.
3. Commit the transaction.  Now that the block group is read only we can commit
   the transaction and be sure the commit roots are consistent with the latest
   changes to the block group.
4. Relocate the blocks in the block group.  This is slightly different for
   metadata versus data block groups.
  a. For data, we use both the `MOVE_DATA_EXTENTS` and `UPDATE_DATA_PTRS` stages
     of the relocation, as described above in the `reloc_inode` section.  The
     first stage simply copies the data to it's new location.  The second stage
     is `UPDATE_DATA_PTRS` and behaves essentially the same way as metadata
     relocation, except we're updating the leaf's with the new location of the
     data.
  b. For metadata, we simply find any blocks and relocate them out of the block
     group, and we then go back and update the parents of the relocated blocks
     to point at the newest block.  This is done
