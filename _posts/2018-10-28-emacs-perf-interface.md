---
layout: post
title: "Emacs' Perf interface"
date: 2018-10-28 11:00:00 +0530
categories: Productivity
---

The combination of <kbd>perf probe -a</kbd> (adding custom probe points) and
<kbd>perf record -e event -g -- workload</kbd> (recording call
traces when the specified probe point is hit along with values of variables)
allowed me to extract information about the state of the kernel without going
through the painfully slow *edit, compile, reboot, test* cycle. 

However, the process of adding custom probe points involved typing away
tedious & boring command lines. For example, consider the following function,

     1 STATIC ssize_t
     2 xfs_file_read_iter(
     3         struct kiocb            *iocb,
     4         struct iov_iter         *to)
     5 {
     6         struct inode            *inode = file_inode(iocb->ki_filp);
     7         struct xfs_mount        *mp = XFS_I(inode)->i_mount;
     8         ssize_t                 ret = 0;
     9 
    10         XFS_STATS_INC(mp, xs_read_calls);
    11 
    12         if (XFS_FORCED_SHUTDOWN(mp))
    13                 return -EIO;
    14 
    15         if (IS_DAX(inode))
    16                 ret = xfs_file_dax_read(iocb, to);
    17         else if (iocb->ki_flags & IOCB_DIRECT)
    18                 ret = xfs_file_dio_aio_read(iocb, to);
    19         else
    20                 ret = xfs_file_buffered_aio_read(iocb, to);
    21 
    22         if (ret > 0)
    23                 XFS_STATS_ADD(mp, xs_read_bytes, ret);
    24         return ret;
    25 }

Lets say that we would want to place a kprobe at line number 20 i.e. just
before the invocation of the function
<kbd>xfs_file_buffered_aio_read()</kbd>. Also, before placing the kprobe we
would want to know the variables that can be recorded by such a kprobe. To
achieve this, we have to first invoke <kbd>perf probe -L</kbd> command to
figure out offset inside <kbd>xfs_file_read_iter()</kbd> to which line number
20 corresponds to.

    $ perf probe -L xfs_file_read_iter --vmlinux=/root/disk-imgs/junk/build/btrfs-next//vmlinux
    <xfs_file_read_iter@/root/repos/linux/fs/xfs/xfs_file.c:0>
          0  xfs_file_read_iter(
                    struct kiocb            *iocb,
                    struct iov_iter         *to)
          3  {
          4         struct inode            *inode = file_inode(iocb->ki_filp);
          5         struct xfs_mount        *mp = XFS_I(inode)->i_mount;
                    ssize_t                 ret = 0;

          8         XFS_STATS_INC(mp, xs_read_calls);

         10         if (XFS_FORCED_SHUTDOWN(mp))
         11                 return -EIO;

         13         if (IS_DAX(inode))
         14                 ret = xfs_file_dax_read(iocb, to);
         15         else if (iocb->ki_flags & IOCB_DIRECT)
         16                 ret = xfs_file_dio_aio_read(iocb, to);
                    else
         18                 ret = xfs_file_buffered_aio_read(iocb, to);

         20         if (ret > 0)
         21                 XFS_STATS_ADD(mp, xs_read_bytes, ret);
                    return ret;
         23  }
		 
As shown in the above output `18` is the offset we were looking for.

Next, we would have to use <kbd>perf probe -V</kbd> command to list the
variables that can be recorded at offset 18 inside the function.

    $ perf probe -q -V xfs_file_read_iter:18 --vmlinux=/root/disk-imgs/junk/build/btrfs-next//vmlinux
    Available variables at xfs_file_read_iter:18
            @<xfs_file_read_iter+168>
                    ssize_t ret
                    struct iov_iter*        to
                    struct kiocb*   iocb
                    struct xfs_mount*       mp
					
The above output indicates that the values of variables <kbd>ret</kbd>,
<kbd>to</kbd>, <kbd>iocb</kbd> and <kbd>mp</kbd> can be recorded from offset
18 of the function.

Finally, we are now ready to place a kprobe at the desired location in the
function.

    $ perf probe -a 'xfs_file_read_iter:18 pos=iocb->ki_pos:u64 mp->m_bsize:s32 iocb:x64' --vmlinux=/root/disk-imgs/junk/build/btrfs-next/vmlinux
    Added new event:
      probe:xfs_file_read_iter (on xfs_file_read_iter:18 with pos=iocb->ki_pos:u64 m_bsize=mp->m_bsize:s32 iocb:x64)

    You can now use it in all perf tools, such as:

            perf record -e probe:xfs_file_read_iter -aR sleep 1

			
In the above command line, we have specified that a kprobe be placed at offset
18 of the function <kbd>xfs_file_read_iter()</kbd> and that the kernel should
collect the values of <kbd>iocb->ki_pos</kbd>, <kbd>mp->m_bsize</kbd> and
<kbd>iocb</kbd> every time the kprobe is hit.

The above mentioned steps needs to be executed for all the interested kernel
functions and the resulting probes must be passed to the <kbd>perf
record</kbd> command line as shown below,

    $ perf record -e probe:xfs_file_read_iter -g -- xfs_io -f -c 'pread 0 64k' /mnt/file-0.bin

As I had mentioned at the beginning of this article, the above process is
quite tedious. In order to take away the boring parts of the above process, I
ended up writing [Emacs lisp
code](https://github.com/chandanr/emacs-config/blob/master/perf.el "perf.el")
to interact with the perf command.

Placing the cursor on a function name and typing <kbd>C-c k p l</kbd> now
executes <kbd>perf probe -L</kbd> command.
![perf probe -l]({{ site.url }}/images/emacs-perf-interface/perf-probe-l.png "perf probe -l")

Typing <kbd>C-c k p a</kbd> now pops up a buffer listing the variables that
can be recorded from a specific offset of a function.

![perf probe -a]({{ site.url }}/images/emacs-perf-interface/perf-probe-a-0.png "perf probe -a")

The following keybindings are active in this buffer,

| a       | Add a variable                                              |
| c       | Change enable/disable status                                |
| r       | Rename variable                                             |
| t       | Provide a perf data type (u8, u16, u32, s8, s16, s32, etc). |
| C-c C-c | Build and execute <kbd>perf probe -a</kbd> command.         |

The following is the result obtained after executing some of the above
mentioned keybindings,
![perf probe -a]({{ site.url }}/images/emacs-perf-interface/perf-probe-a-1.png "perf probe -a")

Typing <kbd>C-c C-c</kbd> builds and executes the <kbd>perf probe -a</kbd>
command line.
![perf probe -a]({{ site.url }}/images/emacs-perf-interface/perf-probe-a-2.png "perf probe -a")

Apart from support for the above listed facilities, <kbd>C-c k p d</kbd>
provides an auto-completed list of probe points from which one of them could
be deleted.
