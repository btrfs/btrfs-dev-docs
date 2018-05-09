Developer documentation of btrfs
================================

Collection of documents that describe internals of the BTRFS filesystem. This
is work in progress and not complete.  The target audience are developers and
not users (refer to btrfs manual pages).

Quick start:

* collect your notes about an area you understand, have reviewed or have documented already
* start a whole structure of the docs, chapters, sections, ...
* you can skip fancy formatting for now
* document how things are implemented, the main areas, operations
* give high-level overview

Examples of topics to cover:

* design
  * how data are organized in b-trees, shadowin/clones etc
  * backreferences
  * physical and logical addressing (aka. chunk tree mappings)
  * data structures
  * free space tracking
  * purpose of various trees
* implementation
  * interaction with pagecache (data and metadata), extents vs pages
  * delayed refs
  * orphan files
  * locking
  * compression write out
  * subvolumes from the inside, radix tree, dentry lookups
  * space reservations
  * transaction commit phases
  * how balance, scrub, dev-replace work
* interactions with other core subsystems
  * ioctl interface
  * VFS: xattr, acl
  * workqueues
  * mount

Workflow:

* initially clone the repository
* create a branch, start a new document or edit existing
* send pull request

Note: master branch is protected and should never be rebased, everything
should go through pull/merge requests. We don't need sign-off, etc. For
changelog, a short description is ok.
