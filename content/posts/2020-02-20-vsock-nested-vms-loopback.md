---
title: "AF_VSOCK: nested VMs and loopback support available"
subtitle: "Recent updates in Linux 5.5 and Linux 5.6"
date: 2020-02-25T20:30:21+01:00
tags: ["vsock", "linux", "qemu", "net", "conference", "talk"]
type: post
---
During the last
[KVM Forum 2019]({{< relref "posts/2019-11-08-kvmforum-2019-vsock" >}}),
we discussed some next steps and several requests came from the audience.

In the last months, we worked on that and recent Linux releases contain
interesting new features that we will describe in this blog post:
* [Nested VMs support](#nested-vms), available in Linux 5.5
* [Local communication support](#local-communication), available in Linux 5.6

## DevConf.CZ 2020
These updates and an introduction to AF\_VSOCK were presented at
**DevConf.CZ 2020** during the
"[VSOCK: VM↔host socket with minimal configuration](https://devconfcz2020a.sched.com/event/YOwb/vsock-vm-host-socket-with-minimal-configuration)" talk.
[Slides](https://static.sched.com/hosted_files/devconfcz2020a/b1/DevConf.CZ_2020_vsock_v1.1.pdf) and [recording](https://youtu.be/R5DQWdPUQSY) are available.

<!--more-->

## Nested VMs
Before Linux 5.5, the AF\_VSOCK core supported only one transport loaded at
run time. That was a limit for nested VMs, because we need multiple transports
loaded together.

### Types of transport
Under the AF\_VSOCK core, that provides the socket interface to the user space
applications, we have several transports that implement the communication
channel between guest and host.

{{< figure src="/img/2020-02-20-vsock-nested-vms-loopback/vsock_transports.png" title="vsock transports" width="300px" >}}

These transports depend on the hypervisor and we can put them in two groups:

* **H2G** (host to guest) transports: they run in the host and
  usually they provide the device emulation; currently we have *vhost* and *vmci*
  transports.
* **G2H** (guest to host) transports: they run in the guest and usually they are
  device drivers; currently we have *virtio, vmci,* and *hyperv* transports.

### Multi-transports
In a nested VM environment, we need to load both G2H and H2G transports
together in the L1 guest, for this reason, we implemented the multi-transports
support to use vsock through **nested VMs**.

{{< figure src="/img/2020-02-20-vsock-nested-vms-loopback/vsock_nested_vms.png" title="vsock and nested VMs" width="300px" >}}

Starting from Linux 5.5, the AF\_VSOCK can handle two types of transports
loaded together at runtime:
* **H2G** transport, to communicate with the guest
* **G2H** transport, to communicate with the host.

So in the QEMU/KVM environment, the L1 guest will load both *virtio-transport*,
to communicate with L0, and *vhost-transport* to communicate with L2.

## Local Communication

Another feature recently added is the possibility to communicate locally on
the same host.
This feature, suggested by [Richard WM Jones](https://rwmj.wordpress.com/),
can be very useful for testing and debugging applications that use AF\_VSOCK
without running VMs.

Linux 5.6 introduces a new transport called *vsock-loopback*, and a new well
know CID for local communication: **VMADDR\_CID\_LOCAL (1)**.
It’s a special CID to direct packets to the same host that generated them.

{{< figure src="/img/2020-02-20-vsock-nested-vms-loopback/vsock_loopback.png" title="vsock loopback" width="300px" >}}

Other CIDs can be used for the same purpose, but it's preferable to use
VMADDR\_CID\_LOCAL:
* Local Guest CID
  * if G2H is loaded (e.g. running in a VM)
* VMADDR\_CID\_HOST (2)
  * if H2G is loaded and G2H is not loaded (e.g. running on L0).
    If G2H is also loaded, then VMADDR\_CID\_HOST is used to reach the host

Richard recently used the vsock local communication to implement a [regression
test](https://github.com/libguestfs/nbdkit/commit/5e4745641bb4676f607fdb3f8750dbf6e9516877)
test for [nbdkit/libnbd vsock support](https://rwmj.wordpress.com/2019/10/21/nbd-over-af_vsock/), using the new [VMADDR\_CID\_LOCAL](https://github.com/libguestfs/nbdkit/commit/01ce88f56f93afd577a8e3b2cc28d825693c8db2).

### Example
```shell
# Listening on port 1234 using ncat(1)
l0$ nc --vsock -l 1234

# Connecting to the local host using VMADDR_CID_LOCAL (1)
l0$ nc --vsock 1 1234
```

## Patches

* [[PATCH net-next v2 00/15] vsock: add multi-transports support](https://patchwork.ozlabs.org/cover/1194660/)
* [[PATCH net-next v2 0/6] vsock: add local transport support](https://patchwork.ozlabs.org/cover/1207012/)


