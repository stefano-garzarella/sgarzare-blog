---
title: "QEMU 4.0 boots uncompressed Linux x86_64 kernel"
date: 2019-08-23T15:26:54+02:00
tags: ["qemu", "linux", "boot"]
type: post
---

**QEMU 4.0** is now able to boot directly into the **uncompressed Linux x86_64 kernel binary** with minimal firmware involvement using the **PVH entry point** defined in the [x86/HVM direct boot ABI](https://xenbits.xen.org/docs/unstable/misc/pvh.html). (`CONFIG_PVH=y` must be enabled in the Linux config file).


The x86/HVM direct boot ABI was initially developed for Xen guests, but with [latest changes in both QEMU and Linux](#patches), QEMU is able to use that same entry point for booting KVM guests.

<!--more-->

## Prerequisites

* QEMU >= 4.0
* Linux kernel >= 4.21
  * `CONFIG_PVH=y` enabled
  * `vmlinux` uncompressed image


## How to use
To boot the PVH kernel image, you can use the `-kernel` parameter specifying the path to the `vmlinux` image.
```
qemu-system-x86_64 -machine q35,accel=kvm \
    -kernel /path/to/vmlinux \
    -drive file=/path/to/rootfs.ext2,if=virtio,format=raw \
    -append 'root=/dev/vda console=ttyS0' -vga none -display none \
    -serial mon:stdio
```

The `-initrd` and `-append` parameters are also supported as for compressed images.


### Details
QEMU will automatically recognize if the `vmlinux` image has the PVH entry point and it will use SeaBIOS with the new `pvh.bin` optionrom to load the uncompressed image into the guest VM.

As an alternative, [qboot](https://github.com/bonzini/qboot) can be used to load the PVH image.


### Performance
Perf script and patches used to measure boot time: https://github.com/stefano-garzarella/qemu-boot-time.

*The following values are expressed in milliseconds [ms]*

* QEMU (q35 machine) + SeaBIOS + bzImage
  * `qemu_init_end`: 36.072056
  * `linux_start_kernel`: 114.669522 (+78.597466)
  * `linux_start_user`: 191.748567 (+77.079045)

* **QEMU (q35 machine) + SeaBIOS + vmlinux(PVH)**
  * `qemu_init_end`: 51.588200
  * `linux_start_kernel`: 62.124665 (+10.536465)
  * `linux_start_user`: 139.460582 (+77.335917)

* QEMU (q35 machine) + qboot + bzImage
  * qemu_init_end: 36.443638
  * linux_start_kernel: 106.73115 (+70.287512)
  * linux_start_user: 184.575531 (+77.844381)

* QEMU (q35 machine) + qboot + vmlinux(PVH)
  * qemu_init_end: 51.877656
  * linux_start_kernel: 56.710735 (+4.833079)
  * linux_start_user: 133.808972 (+77.098237)

* Tracepoints:
  * `qemu_init_end`: first kvm_entry (i.e. QEMU initialization has finished)
  * `linux_start_kernel`: first entry of the Linux kernel (`start_kernel()`)
  * `linux_start_user`: before starting the init process


## Patches

Linux patches merged upstream in Linux 4.21:

* https://lkml.org/lkml/2018/12/14/1330

QEMU patches merged upstream in QEMU 4.0:

* https://www.mail-archive.com/qemu-devel@nongnu.org/msg587312.html
* https://www.mail-archive.com/qemu-devel@nongnu.org/msg588561.html

qboot patches merged upstream:

* https://github.com/bonzini/qboot/pull/17
* https://github.com/bonzini/qboot/pull/18
