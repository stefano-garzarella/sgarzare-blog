---
title: "KVM Forum 2019: virtio-vsock in QEMU, Firecracker and Linux"
subtitle: "Status, Performance and Challenges"
date: 2019-11-09T18:45:25+01:00
tags: ["vsock", "linux", "qemu", "net", "conference", "talk"]
---

[Slides](https://static.sched.com/hosted_files/kvmforum2019/50/KVMForum_2019_virtio_vsock_Andra_Paraschiv_Stefano_Garzarella_v1.3.pdf) and
[recording](https://youtu.be/LFqz-VZPhFE) are available for the "[virtio-vsock in QEMU,
Firecracker and Linux: Status, Performance and Challenges](https://sched.co/TmwK)"
talk that Andra Paraschiv and I presented at
[KVM Forum 2019](https://events19.linuxfoundation.org/events/kvm-forum-2019/).
This was the 13th edition of the KVM Forum conference. It took place in Lyon,
France in October 2019.

We talked about the current status and future works of VSOCK drivers in Linux
and how Firecracker and QEMU provides the virtio-vsock device.

<!--more-->

## Summary

Initially, Andra gave an overview of VSOCK, she described the state of the art,
and the key features:

* it is very simple to configure, the host assigns an unique CID (Context-ID) to
  each guest, and no configuration is needed inside the guest;

* it provides AF\_VSOCK address family, allowing user space application in the
  host and guest to communicate using standard POSIX Socket API (e.g.
  bind, listen, accept, connect, send, recv, etc.)

Andra also described common use cases for VSOCK, such as guest agents
(clipboard sharing, remote console, etc.), network applications using
SOCK\_STREAM, and services provided by the hypervisor to the guests.

Going into the implementation details, Andra explained how the device in the
guest communicates with the vhost backend in the host, exchanging data and
events (i.e. ioeventfd, irqfd).

### Firecracker

Focusing on Firecracker, Andra gave a brief overview on this new VMM (Virtual
Machine Monitor) written in Rust and she explained why, in the v0.18.0 release,
they switched from the experimental vhost-vsock implementation to a vhost-less
solution:

* focus on security impact
* less dependency on host kernel features

This change required a device emulation in Firecracker, that implements
virtio-vsock device model over MMIO. The device is exposed in the host using
UDS (Unix Domain Sockets).

Andra described how Firecracker maps the VSOCK ports on the `uds_path` specified
in the VM configuration:

* Host-Initiated Connections
    * Guest: create an AF\_VSOCK socket and `listen()` on PORT
    * Host: `connect()` to AF\_UNIX at `uds_path`
    * Host: `send()` "CONNECT PORT\n"
    * Guest: `accept()` the new connection

* Guest-Initiated Connections
    * Host: create and `listen()` on an AF\_UNIX socket at `uds_path_PORT`
    * Guest: create an AF\_VSOCK socket and `connect()` to `HOST_CID` and `PORT`
    * Host: `accept()` the new connection

Finally, she showed the performance of this solution, running
[iperf-vsock](https://github.com/stefano-garzarella/iperf-vsock) benchmark,
varying the size of the buffer used in Firecracker to transfer packets
between the virtio-vsock device and the UNIX domain socket. The throughput
on the guest to host path reaches 10 Gbps.

### QEMU

In the second part of the talk, I described the QEMU implementation.
QEMU provides the virtio-vsock device using the vhost-vsock kernel module.

The vsock device in QEMU handles only:

* configuration
    * user or management tool can configure the guest CID
* live-migration
    * connected SOCK\_STREAM sockets become disconnected.
      Applications must handle a connection reset error and should reconnect.
    * guest CID can be not available in the new host because can be
      assigned to another VM. In this case the guest is notified about the
      CID change.

The vhost-vsock kernel module handles the communication with the guest,
providing in-kernel virtio device emulation, to have very high performance and
to interface directly to the host socket layer.
In this way, also host application can directly use POSIX Socket API to
communicate with the guest. So, guest and host applications can be switched
between them, changing only the destination CID.

### virtio-vsock Linux drivers

After that, I told the story of VSOCK in the Linux tree, started in 2013
when the first implementation was merged, and the changes in the last year.

These changes mainly regard fixes, but for the virtio/vhost transports we also
improved the performance with two simple changes released with Linux v5.4:

* reducing the number of credit update messages exchanged
* increasing the size of packets queued in the virtio-vsock device from 4 KB up
  to 64 KB, the maximum packet size handled by virtio-vsock devices.

With these changes we are able to reach ~40 Gbps in the Guest -> Host path,
because the guest can now send up to 64 KB packets directly to the host;
for the Host -> Guest path, we reached ~25 Gbps, because the host is still
using 4 KB buffer preallocated by the guest.

### Tools and languages that support VSOCK

In the last few years, several applications, tools, and languages started to
support VSOCK and I listed them to update the audience:

* Tools:
  * wireshark >= 2.40 [2017-07-19]
  * iproute2 >= 4.15 [2018-01-28]
    * ss
  * tcpdump
    * merged in master [2019-04-16]
  * nmap >= 7.80 [2019-08-10]
    * ncat
  * nbd
    * nbdkit >= 1.15.5 [2019-10-19]
    * libnbd >= 1.1.6 [2019-10-19]
  * iperf-vsock
    * iperf3 fork

* Languages:
  * C
    * glibc >= 2.18 [2013-08-10]
  * Python
    * python >= 3.7 alpha 1 [2017-09-19]
  * Golang
    * https://github.com/mdlayher/vsock
  * Rust
    * libc crate >= 0.2.59 [2019-07-08]
      * struct sockaddr_vm
      * VMADDR_* macros
    * nix crate >= 0.15.0 [2019-08-10]
      * VSOCK supported in the socket API (nix::sys::socket)

### Next steps

Concluding, I went through the next challenges that we are going to face:

* **multi-transport** to use VSOCK in a nested VM environment. because we are
  limited by the fact that the current implementation can handle only one
  transport loaded at run time, so, we can't load virtio_transport and
  vhost_transport together in the L1 guest.
  I already sent some patches upstream
  \[[RFC](https://patchwork.ozlabs.org/cover/1168442/),
  [v1](https://patchwork.ozlabs.org/cover/1181986/)\],
  but they are still in progress.

* **network namespace support** to create independent addressing domains with
  VSOCK socket. This could be useful for partitioning VMs in different
  domains or, in a nested VM environment, to isolate host applications
  from guest applications bound to the same port.

* **virtio-net as a transport for the virtio-vsock** to avoid to re-implement
  features already done in virtio-net, such as mergeable buffers,
  page allocation, small packet handling.


#### From the audience

Other points to be addressed came from the comments we received from the
audience:

* **loopback device** could be very useful for developers to test applications
  that use VSOCK socket. The current implementation support loopback only
  in the guest, but it would be better to support it also in the host, adding
  `VMADDR_CID_LOCAL` special address.

* **VM to VM communication** was asked by several people. Introducing it in the
  VSOCK core could complicate the protocol, the addressing and could require
  some sort of firewall.
  For now we do not have in mind to do it, but I developed a simple user
  space application to solve this issue:
  [vsock-bridge](https://github.com/stefano-garzarella/vsock-bridge).
  In order to improve the performance of this solution, we will consider
  the possibility to add `sendfile(2)` or
  [MSG_ZEROCOPY](https://www.kernel.org/doc/html/v4.15/networking/msg_zerocopy.html)
  support to the AF\_VSOCK core.

* **virtio-vsock windows drivers** is not planned to be addressed, but contributions
  are welcome. Other virtio windows drivers are available in the
  [vm-guest-drivers-windows](https://github.com/virtio-win/kvm-guest-drivers-windows)
  repository.

*Stay tuned!*
