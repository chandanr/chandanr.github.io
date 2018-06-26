---
layout: post
title: "XFS Per-AG space reservation mechanism"
date: 2018-06-26 11:00:00 +0530
categories: Filesystems
---
# Background #
XFS divides the underlying disk space into equal sized chunks called
Allocation Groups (aka AGs). Each AG can function almost independently w.r.t
the rest of the filesystem. For example, each AG has its own Btrees to manage
Free blocks and Inodes.

## Delayed Allocation ##
When applications perform buffered write operations, XFS does not immediately
allocate space on disk for the incoming file data. This also means that XFS
does not allocate space for holding corresponding metadata (E.g. Metadata to
hold mappings of *file offset* to *disk offset*). The incoming file data is
copied into a Per-inode *Page cache* and the `write()` syscall returns back
with a success status. The contents of this Page cache is flushed to disk when
one of the following events occur:
  * Increased memory pressure
  * Userspace invokes `fsync()/sync()` syscalls.

When one of the above listed events occur, XFS starts allocating disk space
for holding both *file data* and the corresponding *metadata*. On successful
space allocation, the contents of the Per-inode *Page cache* are copied to the
corresponding offset on the disk.

For the previously mentioned allocations to succeed, XFS would have reserved
enough disk space for housing both the data and the metadata when servicing
the `write()` syscall (See `xfs_bmapi_reserve_delalloc()`). These space
reservations are *filesystem-wide* i.e. they decrement the free block counter
available at `xfs_mount->m_fdblocks`. If there isn't sufficient free space
available (as indicated by `xfs_mount->m_fdblocks`) XFS ends up returning
-ENOSPC error code.

This mechanism of delaying the allocation of actual disk blocks for file data
written via buffered write mechanism is called *Delayed Allocation*.

# XFS Per-AG space reservation mechanism #
However the previously described *filesystem-wide* space reservation mechanism
falls apart when using the *reflink* feature. First, let us look into the core
data structure used to implement the reflink feature.

## XFS Reflink feature ##
XFS' Reflink feature allows two or more files to share the same data
blocks. Reflink is essentially implemented by having a Per-AG Btree track the
number of files sharing the used disk extents. Each record in the leaves of
reflink/refcount Btree has the following structure,

    struct xfs_refcount_irec {
    	xfs_agblock_t	rc_startblock;
    	xfs_extlen_t	rc_blockcount;
    	xfs_nlink_t	rc_refcount;
    };

`rc_startblock` gives the offset within the AG at which the extent begins,
`rc_blockcount` gives the number of blocks making up the extent and
`rc_refcount` gives the number of inodes/files sharing this extent.

`rc_startblock` being an offset into the AG rather than a filesystem-wide
offset gives rise to a problem.

Lets assume that we have two files `src.bin` and `dst.bin`.

    [root@localhost ~]# ls -id1 /mnt/
    128 /mnt/
    [root@localhost ~]# ls -i1 /mnt/
        131 dir0
    8388736 dir1
    [root@localhost ~]# ls -i /mnt/dir0
    132 src.bin
    [root@localhost ~]# ls -i /mnt/dir1
    8388737 dst.bin

NOTE: XFS spreads directories and the corresponding directory entries across
AGs. This is done to increase I/O bandwidth.

    xfs_db> inode 128
    xfs_db> p
    core.magic = 0x494e
    core.mode = 040755
    core.version = 3
    ...
    u3.sfdir3.list[0].name = "dir0"
    u3.sfdir3.list[0].inumber.i4 = 131
    u3.sfdir3.list[0].filetype = 2
    u3.sfdir3.list[1].namelen = 4
    u3.sfdir3.list[1].offset = 0x70
    u3.sfdir3.list[1].name = "dir1"
    u3.sfdir3.list[1].inumber.i4 = 8388736
    u3.sfdir3.list[1].filetype = 2

    xfs_db> convert inode 131 agno
    0x0 (0)

    xfs_db> convert inode 8388736 agno
    0x1 (1)

The above output shows that `dir0's` inode resides in AG 0 while `dir1's`
inode resides in AG 1.

`src.bin` and `dst.bin` were created using the following commands,

    [root@localhost ~]# xfs_io -f -c 'pwrite 0 512m' /mnt/dir0/src.bin
    wrote 536870912/536870912 bytes at offset 0
    512 MiB, 131072 ops; 0:00:01.32 (387.680 MiB/sec and 99246.0685 ops/sec)
    [root@localhost ~]# sync
    [root@localhost ~]# cp --reflink=always /mnt/dir0/src.bin /mnt/dir1/dst.bin

The `cp` command causes `dst.bin` to share the blocks/extents orignally owned
by `src.bin`. Here is `src.bin's` metadata,

    xfs_db> inode 132
    xfs_db> p
    core.magic = 0x494e
    core.mode = 0100600
    core.version = 3
    core.format = 2 (extents)
    core.nlinkv2 = 1
    core.onlink = 0
    core.projid_lo = 0
    core.projid_hi = 0
    core.uid = 0
    core.gid = 0
    ...
    u3.bmx[0] = [startoff,startblock,blockcount,extentflag]
    0:[0,24,131072,0]

The last two lines print the mapping between *file offset* and *disk offset*
i.e. the zeroth file block is mapped to 24th disk block. Also, the length of
the extent i.e. 131072 blocks (512MiB) is printed as well.

Lets check the contents of `dst.bin`,

    xfs_db> inode 8388737
    xfs_db> p
    core.magic = 0x494e
    core.mode = 0100600
    core.version = 3
    core.format = 2 (extents)
    core.nlinkv2 = 1
    core.onlink = 0
    core.projid_lo = 0
    core.projid_hi = 0
    core.uid = 0
    core.gid = 0
    ...
    u3.bmx[0] = [startoff,startblock,blockcount,extentflag]
    0:[0,24,131072,0]

The last line confirms that both `src.bin` and `dst.bin` share the same blocks
i.e they both have their data starting at 24th disk block. At this instant,
the *Reflink* btree in AG 0 holds the following,

    xfs_db> convert agbno 6 fsblock
    0x6 (6)
    xfs_db> fsblock 6
    xfs_db> type refcntbt
    xfs_db> p
    magic = 0x52334643
    level = 0
    numrecs = 1
    leftsib = null
    rightsib = null
    bno = 48
    lsn = 0x10000000f
    uuid = 7076c22e-68bf-4142-807f-063d031126bc
    owner = 0
    crc = 0xcd863d03 (correct)
    recs[1] = [startblock,blockcount,refcount,cowflag]
    1:[24,131072,2,0]

The last line shows that the extent beginning at 24th block of the AG is being
referred to by two files (notice that the value of the third field is 2).

What happens when one of the two files residing in two different AGs
and sharing the same blocks is written to by a write operation. For example,

    [root@localhost ~]# xfs_io -f -c 'pwrite 64k 64k' -c sync /mnt/dir1/dst.bin
    wrote 65536/65536 bytes at offset 65536
    64 KiB, 16 ops; 0.0003 sec (184.911 MiB/sec and 47337.2781 ops/sec)

    xfs_db> agf 0
    xfs_db> convert agbno 6 fsblock
    0x6 (6)
    xfs_db> fsblock 6
    xfs_db> type refcntbt
    xfs_db> p
    magic = 0x52334643
    level = 0
    numrecs = 2
    leftsib = null
    rightsib = null
    bno = 48
    lsn = 0x10000001c
    uuid = 7076c22e-68bf-4142-807f-063d031126bc
    owner = 0
    crc = 0x88e9b35c (correct)
    recs[1-2] = [startblock,blockcount,refcount,cowflag]
    1:[24,16,2,0] 
    2:[56,131040,2,0]

Here we write 16 blocks worth of data into `dst.bin` at 16th block of the
file. Remember that `dst.bin's` inode is actually present in AG 1. But the
write operation has caused a new entry to be added into the *Reflink* Btree
present in AG 0.

i.e. `AG X's` reflink/refcount Btree could expand due to write operation
performed on files whose inode is stored in another AG but which shares data
blocks originally owned by a file residing in `AG X`. In such cases, a
*filesystem-wide* space reservation wouldn't be sufficient to prevent a ENOSPC
from occuring if `AG X` gets filled up while allocating blocks to hold the
possible new records of reflink tree in `AG X`.

Hence, XFS [reserves space for Reflink/Refcount Btrees on a Per-AG
basis](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3fd129b63fd062a0d8f5d55994a6e98896c20fa7
"3fd129b63fd062a0d8f5d55994a6e98896c20fa7").
