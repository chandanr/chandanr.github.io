---
layout: post
title: "XFS: Shared data blocks"
date: 2018-10-19 11:00:00 +0530
categories: Filesystems
---

# Introduction #
Filesystems generally have on-disk data structures to map a file's offset
range to physical blocks on disk. These mappings are associated with the inode
representing the file.

XFS uses <kbd>struct xfs_bmbt_rec</kbd> to store such a mapping.

    typedef struct xfs_bmbt_rec {
            __be64                  l0, l1;
    } xfs_bmbt_rec_t;

The two members of the above structure are used to encode the *file offset to
disk offset* mapping as shown below:

  * Bits 9-62 of `l0` encode the file offset that is being mapped.
  * Bits 0-8 of `l0` along with bits 21-63 of `l1` encode the disk block.
  * Bits 0-20 of `l1` encode the number of blocks being mapped.

The in-core version of this structure is represented by <kbd>struct
xfs_bmbt_irec</kbd> whose members <kbd>br_startoff</kbd>,
<kbd>br_startblock</kbd> and <kbd>br_blockcount</kbd> which store the *File
offset*, *Disk offset* and *Number of blocks* mapped by this extent.

A file read/write operation will essentially iterate across a sequence of
<kbd>struct xfs_bmbt_irec</kbd> structures until it finds the instance of the
structure which maps the file offset from which the user space application
wants to read/write data. A disk read/write operation is then issued to the
disk offset obtained from the mapping structure.

# Shared Data blocks #

Some of the use cases end up creating files containing the same data. As an
example, consider creating a copy of a large file,

    # ls -sh /mnt/src.bin
    1.0G /mnt/src.bin
    # cp /mnt/src.bin /mnt/dst.bin
    # filefrag -b1 -v /mnt/src.bin
    Filesystem type is: 58465342
    File size of /mnt/src.bin is 1073741824 (1073741824 blocks of 1 bytes)
     ext:     logical_offset:        physical_offset: length:   expected: flags:
       0:        0..1073741823:      98304..1073840127: 1073741824:             last,eof
    /mnt/src.bin: 1 extent found
    [root@localhost mnt]# filefrag -b1 -v /mnt/dst.bin
    Filesystem type is: 58465342
    File size of /mnt/dst.bin is 1073741824 (1073741824 blocks of 1 bytes)
     ext:     logical_offset:        physical_offset: length:   expected: flags:
       0:        0..1073741823: 1073840128..2147581951: 1073741824:             last,eof
    /mnt/dst.bin: 1 extent found

As can be seen from the output of the `filefrag` command, both `src.bin` and
`dst.bin` end up unnecessarily occupying two sets of disk blocks even though
they contain the same data. A real world example would be that of two or more
virtualized guests initially sharing the same disk images.

XFS (via the [reflink
feature](https://chandanr.github.io/filesystems/2018/06/26/xfs-perag-space-resv.html
"Per-AG space reservation")) provides a method to share disk blocks among two
or more files.

# Per-Inode forks #
After being read from the disk, all the `file offset to disk offset` mappings
belonging to the file are stored in what is called the *Data fork* of the
in-core inode. This region of the in-core inode stores one of the following,
  * Inline data for small files.
  * An array of <kbd>struct xfs_bmbt_irec</kbd> structures.
  * The root of an in-core Btree whose leaves contain several <kbd>struct
    xfs_bmbt_irec</kbd> structures.

Essentially the data fork of an inode holds metadata required to obtain the
file's actual content. Similarly there is an *Attribute fork*
which stores the metadata required to access an inode's *Extended attributes*.

## CoW fork ##
To support shared disk blocks, a new fork called the *CoW fork* was added to
the in-core XFS inode. The purpose of this fork is to act as an area where the
new disk mappings (i.e. delayed allocations) can be staged temporarily when a
write is performed on shared blocks of a file.

To understand the requirement for a new Per-inode fork, let us first list the
events that take place when a write operation is performed on a regular file,
1. User-space application invokes buffered write operation.
2. XFS does a delayed extent allocation reservation i.e. a <kbd>struct
   xfs_bmbt_irec</kbd> with <kbd>xfs_bmbt_irec->br_startblock</kbd> set to the
   output of <kbd>nullstartblock()</kbd>.
   OR
   There is nothing to be done if the extent has a valid value for
   <kbd>xfs_bmbt_irec->br_startblock</kbd> i.e. the extent already has disk
   blocks mapped for the file offset range to which user-space is trying to
   write new data to.
3. The file data is copied from user-space pages to the per-inode page-cache
   pages.
4. Later, when these dirty pages are supposed to be written to the disk, the
   extent mappings present in the data fork of the inode are referred to
   figure out the location on the disk.
   If the extent mapping was a *delayed allocation* mapping, XFS allocates
   space on the disk and the corresponding
   <kbd>xfs_bmbt_irec->br_startblock</kbd> is set to the disk offset at which
   the newly allocated space begins.
5. A disk write operation is then initiated.

Writing to a file range that is mapped to shared disk blocks isn't allowed
since the contents of the shared disk blocks are being referred to by atleast
one more file. Hence a new extent allocation must be done for such write
operations. This would mean that XFS would have to follow the "delayed
allocation" procedure used when doing append writes i.e. Reserve space and
replace the existing extent mapping with a new one in the in-core inode's data
fork. However such delayed allocation mappings become valid only when *real
disk space* is later allocated when flushing the dirty pages in the page
cache. If a failure occurs during this phase, the mapping in the in-core inode
would end up being incorrect. The per-inode CoW fork was added to solve this
problem.

[CoW
fork](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3993baeb3c52f497d243a4a3b5510df97b22596b
"3993baeb3c52f497d243a4a3b5510df97b22596b") holds the delayed allocation
extent mappings when data is being written to shared blocks of a file. Once
the dirty pages are flushed to disk, these mappings are removed from the CoW
fork and they replace the corresponding shared mappings in the inode's Data
fork.
