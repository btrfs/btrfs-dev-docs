# Data extent reference counts

## What are bookends?

There are a few references to bookends in the codebase and from developer
emails.  This is what we use to describe what happens to the reference counts
when you split an extent.  Consider the following example file with a single
extent

    0                                  1mib
    +----------------------------------+
    |             1 mib                |
    +----------------------------------+

This is simple, we have 1 extent with 1 reference count that is 1mib in size,
thus taking up 1mib of total disk space.  The inode has 1 file extent

```
        item 7 key (257 EXTENT_DATA 0) itemoff 15733 itemsize 53
                generation 6 type 1 (regular)
                extent data disk byte 13631488 nr 1048576
                extent data offset 0 nr 1048576 ram 1048576
                extent compression 0 (none)
```

and there is 1 reference for that disk bytenr in the extent tree

```
        item 10 key (13631488 EXTENT_ITEM 1048576) itemoff 15918 itemsize 53
                refs 1 gen 6 flags DATA
                extent data backref root FS_TREE objectid 257 offset 0 count 1
```

Now consider we write 4kib into the middle of this existing extent, so the file
now looks like this

    0             512k   512+4k       1mib
    +-------------+------+-------------+
    |     a       | 4kib |     b       |
    +-------------+------+-------------+

While it appears we have 3 extents, and indeed we have 3 file extents for this
inode, we only have 2 actual on disk data extents.  Both `a` and `b`  are what
we call bookends.  They are file extents that point to different ranges in the
original 1mib extent.  To make this clearer, these are the extents for the inode
now

```
        item 7 key (257 EXTENT_DATA 0) itemoff 15733 itemsize 53
                generation 6 type 1 (regular)
                extent data disk byte 13631488 nr 1048576
                extent data offset 0 nr 524288 ram 1048576
                extent compression 0 (none)
        item 8 key (257 EXTENT_DATA 524288) itemoff 15680 itemsize 53
                generation 7 type 1 (regular)
                extent data disk byte 14680064 nr 4096
                extent data offset 0 nr 4096 ram 4096
                extent compression 0 (none)
        item 9 key (257 EXTENT_DATA 528384) itemoff 15627 itemsize 53
                generation 6 type 1 (regular)
                extent data disk byte 13631488 nr 1048576
                extent data offset 528384 nr 520192 ram 1048576
                extent compression 0 (none)
```

and these are the extents in the extent tree

```
        item 10 key (13631488 EXTENT_ITEM 1048576) itemoff 15918 itemsize 53
                refs 2 gen 6 flags DATA
                extent data backref root FS_TREE objectid 257 offset 0 count 2
        item 12 key (14680064 EXTENT_ITEM 4096) itemoff 15841 itemsize 53
                refs 1 gen 7 flags DATA
                extent data backref root FS_TREE objectid 257 offset 524288 count 1
```

The original extent, bytenr 13631488, now has 2 references, and the new 4kib
extent has 1 reference.

## Anatomy of a bookend

Taking the above example and looking at the file extents in the inode for just
the bookends

```
        item 7 key (257 EXTENT_DATA 0) itemoff 15733 itemsize 53
                generation 6 type 1 (regular)
                extent data disk byte 13631488 nr 1048576
                extent data offset 0 nr 524288 ram 1048576
                extent compression 0 (none)
        item 9 key (257 EXTENT_DATA 528384) itemoff 15627 itemsize 53
                generation 6 type 1 (regular)
                extent data disk byte 13631488 nr 1048576
                extent data offset 528384 nr 520192 ram 1048576
                extent compression 0 (none)
```

You see they both have the same `data disk byte` of 13631488.  However the
second extent has a `data offset` of 528384, which is the offset into the
original extent.  The `nr` for the actual disk bytenr is still the same,
1048576, only the `data offset` section has the true length of `520192` which is
the length of the extent itself.

## Handling reference counting

The above example is straightforward.  We split an existing file extent into two
different extents, and we add a reference for the split extent.  Most of this
work is handled in `btrfs_drop_extents`, which is called in a variety of
different instances.

Consider the following example.

```
disk_bytenr = 0
disk_len = 1mib
orig_offset = 0
new_start = 512kib
new_len = 4kib

duplicate the file extent, set the new extent key to 512kib+4kib.

// On the new extent
btrfs_set_file_extent_num_bytes(orig_len - new_start + new_len);
btrfs_set_file_extent_offset(orig_offset + new_start + new_len);
btrfs_inc_extent_ref(disk_bytenr, inode number, orig_offset);

// On the original extent
btrfs_set_file_extent_num_bytes(orig_len - new_start);
```

The only time we drop an extent reference is if we delete an entire extent.

## What data references look like

Consider again the extent reference for the original 1mib extent after we did
the split

```
        item 10 key (13631488 EXTENT_ITEM 1048576) itemoff 15918 itemsize 53
                refs 2 gen 6 flags DATA
                extent data backref root FS_TREE objectid 257 offset 0 count 2
```

This is covered more in backreferences.txt, but lets look at how this works with
the file extent items.  As you can see here we have `objectid 257 offset 0`, but
one of the references isn't actually at offset 0, it's at offset 528384.  This
is another use for the `btrfs_file_extent_offset`.  It doesn't only point to the
offset in the physical location, but helps us determine what the offset was for
our original reference.  If we were to remove the second extent, we would use
this offset to figure out which extent reference we wanted to remove.  Say we
truncated the file down to 528384 bytes and removed that second extent, we would
do something like the following to remove our reference

```
disk_bytenr = btrfs_file_extent_disk_bytenr(fi);
disk_bytes = btrfs_file_extent_disk_num_bytes(fi);
offset = key.offset - btrfs_file_extent_offset(offset);
btrfs_free_extent(disk_bytenr, disk_bytes, key.objectid (inode number), offset);
```

As you can see, we never modify the `disk_bytenr` or the `disk_num_bytes` in the
file extent, those always refer to the extent that we reference.  We simply
modify the `file_extent_offset` based on where it is in relation to the original
extent, and then calculate the proper information when we modify the reference
counts.

## What does this mean practically?

First, it makes the extent tree relatively simple.  We don't end up with a lot
of extra items every time we split an extent, so our extent tree stays very
small in the face of a lot of data fragmentation.

This also makes the rules and code surrounding splitting less hairy.  We just
need to figure out the disk bytenr and make the appropriate inc/dec operation.
The only complicated part is the actual splitting of the file extent itself.

There's a huge con however, and it's that overwrite workloads potentially waste
a lot of space.

Consider the following worst case

    +------------+
    |    1mib    |
    +------------+
    +-----------+
    |   1m-4k   |
    +-----------+
    +----------+
    |   1m-8k  |
    +----------+
    +---------+
    |  1m-12k |
    +---------+

The disk usage of this write pattern results in ~4mib of space used on disk.
Each subsequent write leaves 4kib of the previous extent around, so it is not
freed.

For this reason, among others, we recommend using NODATACOW for over write
workloads, as the space usage can really balloon up.
