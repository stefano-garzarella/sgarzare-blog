---
title: "AF_VSOCK: network namespace support is here"
subtitle: "New in Linux 7.0"
date: 2026-02-11T18:00:00+01:00
tags: ["vsock", "linux", "qemu", "net"]
type: post
---

A missing piece for **AF\_VSOCK** in the Linux kernel has been
**network namespace** support. We discussed it as a future challenge during the
[KVM Forum 2019]({{< relref "posts/2019-11-08-kvmforum-2019-vsock" >}}) talk
and it was mentioned in several conference discussions since then.

I started working on namespace support back in
[2019](https://lore.kernel.org/netdev/20200116172428.311437-1-sgarzare@redhat.com/),
but never had the chance to complete it. Last year, **Bobby Eshleman** (Meta)
restarted the effort and drove it through 16 revisions of the patch series.
**Daniel Berrangé**, **Michael S. Tsirkin**, **Paolo Abeni**, and I contributed with
reviews and suggestions that shaped the current user API.
The result has been merged into `net-next` and will be available in
**Linux 7.0**.

<!--more-->

## Background

Network namespaces are a fundamental building block for containers in Linux.
They provide isolation of the network stack, so each namespace has its own
interfaces, routing tables, and sockets.

Before Linux 7.0, AF\_VSOCK was not namespace-aware. All vsock sockets lived in
the same global space, regardless of the network namespace they were created
in. This caused two problems:

* **No isolation**: a VM started inside a network namespace (or container)
  was reachable via vsock from any other namespace on the host, breaking the
  isolation that containers expect.
* **No CID reuse**: since CIDs were global, two VMs in different namespaces
  could not use the same CID, even if they were completely isolated from
  each other at the network level.

## Design

The new implementation introduces two modes, configured per network namespace:

* **global**: CIDs are shared across namespaces. This is the original behavior
  and the default, so existing setups continue to work without any change.
* **local**: namespaces are completely isolated. Sockets in a local-mode
  namespace can only communicate with other sockets in the same namespace.

Two sysctl knobs are available since Linux 7.0:

* `/proc/sys/net/vsock/child_ns_mode`: the parent namespace uses this to set
  the mode that new child namespaces will inherit. Accepts `global` or `local`.
* `/proc/sys/net/vsock/ns_mode`: read-only, shows the mode of the current
  namespace. The mode is immutable after namespace creation.

This design ensures backward compatibility: the default is `global`, matching
the previous behavior. Namespace isolation is opt-in.

Each namespace gets its mode from the parent's `child_ns_mode` at
creation time. Once set, the namespace's `ns_mode` is immutable: every
socket and VM in that namespace follows it. Changing `child_ns_mode`
in the parent only affects future child namespaces, not existing ones.

## Supported vsock transports

This series adds namespace support to two transports:

* **vhost-vsock**: host-to-guest (H2G) transport, emulates the virtio-vsock
  device for KVM guests
* **vsock-loopback**: local transport, useful for testing and debugging
  without running VMs

The missing transports are the guest-to-host (G2H) ones (virtio, hyperv, vmci).
These run in the guest as device drivers, and we currently don't have a way to
assign a vsock device to a specific namespace, since vsock devices are not
standard network devices. For now, they operate in `global` mode, so they are reachable from
any `global` namespace, but not from `local` namespaces. This means that
sockets in a `local` namespace cannot communicate with the host through
these transports. We plan to work on that in the future.

## Examples

### Loopback

In the following examples, the commands without a namespace prefix run in the
initial network namespace (init\_netns), which is the default namespace where
all processes start. The init\_netns is always in `global` mode.

These examples use the vsock loopback device for local communication,
without any VM involved.

Make sure the `vsock_loopback` kernel module is loaded:
```shell
$ sudo modprobe vsock_loopback
```

#### Namespace isolation with unshare

##### Global mode (default)

By default, `child_ns_mode` is set to `global`. This is the same behavior
as before Linux 7.0: vsock sockets are shared across namespaces.

A listener started in a new namespace is reachable from the init\_netns
using the loopback CID (`VMADDR_CID_LOCAL = 1`):
```shell
$ echo global | sudo tee /proc/sys/net/vsock/child_ns_mode
$ unshare --user --net nc --vsock -l 1234 &
$ nc --vsock 1 1234
# reachable - global mode, no isolation
```

##### Local mode

Setting `child_ns_mode` to `local` enables isolation. New namespaces will
have their own vsock space:
```shell
$ echo local | sudo tee /proc/sys/net/vsock/child_ns_mode
```

Now a listener in a new namespace is not reachable from the init\_netns:
```shell
$ unshare --user --net nc --vsock -l 1234 &
$ nc --vsock 1 1234
Ncat: Connection reset by peer.
```

#### Namespace isolation with ip netns

The same can be done with `ip netns`, which requires root (or `CAP_SYS_ADMIN`).

First, create a `global` namespace and check its mode:
```shell
$ echo global | sudo tee /proc/sys/net/vsock/child_ns_mode
$ sudo ip netns add vsock_ns_global
$ sudo ip netns exec vsock_ns_global cat /proc/sys/net/vsock/ns_mode
global
```

A listener in the `global` namespace is reachable from the init\_netns:
```shell
$ sudo ip netns exec vsock_ns_global nc --vsock -l 1234 &
$ nc --vsock 1 1234
# reachable - global mode, no isolation
```

Now create a `local` namespace and check its mode:
```shell
$ echo local | sudo tee /proc/sys/net/vsock/child_ns_mode
$ sudo ip netns add vsock_ns_local
$ sudo ip netns exec vsock_ns_local cat /proc/sys/net/vsock/ns_mode
local
```

A listener in the `local` namespace is not reachable from the init\_netns:
```shell
$ sudo ip netns exec vsock_ns_local nc --vsock -l 1234 &
$ nc --vsock 1 1234
Ncat: Connection reset by peer.
```

But communication within the same namespace still works:
```shell
$ sudo ip netns exec vsock_ns_local nc --vsock 1 1234
# reachable - same namespace
```

#### Container isolation with podman

Since `podman` creates a network namespace for each container by default,
vsock namespace support applies to containers as well.

First, build a Fedora-based image with `ncat` installed:
```shell
$ podman build -t fedora-ncat - <<< "FROM fedora
RUN dnf -y install nmap-ncat"
```

With the default `global` mode, two containers share the same vsock space.
A listener in one container is reachable from another `global` container:
```shell
$ echo global | sudo tee /proc/sys/net/vsock/child_ns_mode
$ podman run --rm --init -d fedora-ncat sh -c "echo hello world | nc --vsock -l 1234"
$ podman run --rm --init -it fedora-ncat nc --vsock 1 1234
hello world
```

With `local` mode, each container gets its own isolated vsock namespace:
```shell
$ echo local | sudo tee /proc/sys/net/vsock/child_ns_mode
$ podman run --rm --init -d fedora-ncat sh -c "echo hello world | nc --vsock -l 1234"
$ podman run --rm --init -it fedora-ncat nc --vsock 1 1234
Ncat: Connection reset by peer.
# containers are isolated from each other
```

### VMs with QEMU

The vhost-vsock H2G transport exposes the `/dev/vhost-vsock` device, which
QEMU opens at VM startup to emulate the virtio-vsock device for the guest.
Since namespace support applies to this transport, VMs inherit the namespace
mode as well.

In the following examples, we reuse the `vsock_ns_global` and `vsock_ns_local`
namespaces created in the previous section.

#### Global mode

With `global` mode, the VM started in a `global` namespace is reachable
from any other `global` namespace, including the init\_netns:
```shell
$ sudo ip netns exec vsock_ns_global \
    qemu-system-x86_64 -m 1G -M q35,accel=kvm \
    -drive file=guest.qcow2,if=virtio,snapshot=on \
    -device vhost-vsock-pci,guest-cid=42

# start a listener in the guest (global namespace)
guest_global$ nc --vsock -l 1234

# from the init_netns (global) - reachable
$ nc --vsock 42 1234
```

#### Local mode

With `local` mode, the VM is only reachable from within the same namespace:
```shell
$ sudo ip netns exec vsock_ns_local \
    qemu-system-x86_64 -m 1G -M q35,accel=kvm \
    -drive file=guest.qcow2,if=virtio,snapshot=on \
    -device vhost-vsock-pci,guest-cid=42

# start a listener in the guest (local namespace)
guest_local$ nc --vsock -l 1234

# from the init_netns (global) - isolated
$ nc --vsock 42 1234
Ncat: Connection reset by peer.

# from the same namespace - reachable
$ sudo ip netns exec vsock_ns_local nc --vsock 42 1234
```

#### Guest-to-host (G2H) behavior

As mentioned in the [Supported vsock transports](#supported-vsock-transports)
section, the G2H virtio transport does not support namespaces yet. The
virtio-vsock device in the guest always operates in `global` mode, so
only sockets in `global` namespaces can communicate with the host.

Using the VM started in `vsock_ns_global`, a listener in the guest's
init\_netns is reachable from the host:
```shell
# start a listener in the guest
guest_global$ nc --vsock -l 1234

# from the host - reachable
$ nc --vsock 42 1234
```

But a listener started in a `local` namespace inside the guest is not
reachable from the host:
```shell
# create a local namespace in the guest and start a listener
guest_global$ echo local | sudo tee /proc/sys/net/vsock/child_ns_mode
guest_global$ unshare --user --net nc --vsock -l 1234

# from the host - isolated
$ nc --vsock 42 1234
Ncat: Connection reset by peer.
```

#### CID reuse

Note that we used the same CID (`42`) in both examples without turning off
the first VM. This is possible because the second VM is in a `local`
namespace, so its CID space is isolated. With `global` mode, QEMU would
fail to start the second VM because the CID is already in use.

## Patches

* [[PATCH net-next v16 00/12] vsock: add namespace support to vhost-vsock and loopback](https://lore.kernel.org/netdev/20260121-vsock-vmtest-v16-0-2859a7512097@meta.com/)
