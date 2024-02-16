---
title: "QEMU 4.2 mmap(2)s kernel and initrd"
date: 2019-12-22T17:46:44+01:00
tags: [linux", "qemu", "boot", "microvm"]
type: post
---

In order to save memory and boot time, **QEMU 4.2** and later versions are able to
mmap(2) the kernel and initrd specified with `-kernel` and `-initrd` parameters.
This approach allows us to avoid reading and copying them into a buffer,
saving memory and time.

The memory pages that contain kernel and initrd are shared between
multiple VMs using the same kernel and initrd images.
So, when many VMs are launched we can save memory by sharing pages, and save
time by avoiding reading them each time from the disk.

<!--more-->

This feature is automatically used for some targets with ELF kernels
(e.g. x86\_64 vmlinux ELF image with
[PVH entry point]({{< relref "posts/2019-08-23-qemu-linux-kernel-pvh" >}})),
but it is not available with compressed kernel images (e.g. bzImage).

## Patches

The
[patches](https://patchew.org/QEMU/20190724143105.307042-1-sgarzare@redhat.com/)
that implement this feature are merged upstream and released with
[QEMU 4.2](https://wiki.qemu.org/ChangeLog/4.2#Miscellaneous).

The main change was to map kernel and initrd into the memory, instead of
reading them, using `g_mapped_file_*()` APIs.

## Benchmarks
We measured the memory footprint and the boot time using a standard QEMU
build (`qemu-system-x86_64`) with a
[PVH kernel]({{< relref "posts/2019-08-23-qemu-linux-kernel-pvh" >}})
and initrd (cpio):

* Initrd size: 3.0M
* Kernel (vmlinux)
  * image size: 28M
  * sections size [`size -A -d vmlinux`]:  18.9M

Julio Montes did a very good analysis and he posted the results here:
https://www.mail-archive.com/qemu-devel@nongnu.org/msg633168.html

### Memory footprint
We used [smem](https://www.selenic.com/smem/) to measure
[USS and PSS](https://www.golinuxcloud.com/check-memory-usage-per-process-linux/):

* USS (Unique Set Size): amount of memory that is committed to physical memory
  and is unique to a process; it is not shared with any other. It is the
  amount of memory that would be freed if the process were to terminate.
* PSS (Proportional Set Size): This splits the accounting of shared pages that
  are committed to physical memory between all the processes that have them
  mapped.

```shell
$ smem -k | grep "PID\|$(pidof qemu-system-x86_64)"
  PID User     Command                         Swap      USS      PSS      RSS
24833 qemu     /usr/bin/qemu-system-x86_64        0    71.6M    74.3M    105.2
```

This is the memory footprint analysis, increasing the number of QEMU instances:
```shell
                           Memory footprint [MB]
     QEMU             before                 after
 # instances        USS     PSS           USS     PSS
      1           102.0   105.8         102.3   106.2
      2            94.6   101.2          72.3    90.1
      4            94.1    98.0          72.0    81.5
      8            94.0    96.2          71.8    76.9
     16            93.9    95.1          71.6    74.3
```

### Boot time
We measured the boot time using the
[qemu-boot-time](https://github.com/stefano-garzarella/qemu-boot-time)
scripts described in this
[post]( {{< relref "posts/2019-08-24-qemu-linux-boot-time" >}}).

This is the boot time analysis:
```shell
                                   Boot time [ms]
                          before                  after
 # trace points
 qemu_init_end:           63.85                   55.91
 linux_start_kernel:      82.11 (+18.26)          74.51 (+18.60)
 linux_start_user:       169.94 (+87.83)         159.06 (+84.56)
```

## Conclusions
Mapping into memory the kernel and initrd images allowed us to save about 20 MB
of memory when multiple VMs are started and allowed us to speed up the boot by
about 10 milliseconds.

**Note:** both gains are strictly related to images size.
