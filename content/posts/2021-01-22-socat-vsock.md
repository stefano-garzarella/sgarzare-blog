---
title: "SOCAT now supports AF_VSOCK"
subtitle: ""
date: 2021-01-22T15:16:04+01:00
tags: ["vsock", "linux", "qemu", "net", "conference", "talk", "socat"]
---

[SOCAT](http://www.dest-unreach.org/isocat/)
is a CLI utility which enables the concatenation
of two sockets together.
It establishes two bidirectional byte streams and transfers data between them.

`socat` supports several address types (e.g. TCP, UDP, UNIX domain sockets, etc.)
to construct the streams. The latest version **1.7.4**, released earlier this
year [2021-01-04], supports also AF_VSOCK addresses:
- [VSOCK-LISTEN](http://www.dest-unreach.org/socat/doc/socat.html#ADDRESS_VSOCK_LISTEN):`<port>`
  - Listen on `port` and accepts a VSOCK connection.

- [VSOCK-CONNECT](http://www.dest-unreach.org/socat/doc/socat.html#ADDRESS_VSOCK_CONNECT):`<cid>:<port>`
  - Establishes a VSOCK stream connection to the specified `cid` and `port`.


## FOSDEM 2021

If you are interested on VSOCK, I'll talk witn Andra Paraschiv (AWS) about it
at FOSDEM 2021.
The talk is titled
[Leveraging virtio-vsock in the cloud and containers](https://fosdem.org/2021/schedule/event/vai_virtio_vsock)
and it's scheduled for Saturday, February 6th 2021 at 11:30 AM (CET).

We will show cool VSOCK use cases and some demos about developing, debugging,
and measuring the VSOCK performance, including `socat` demos.

<!--more-->

## Examples

`socat` could be very useful for concatenating and redirecting sockets.
In this section we will see some examples.

Examples below refer to a guest with `CID 42` that we created using
[virt-builder](https://libguestfs.org/virt-builder.1.html)
and
[virt-install](https://virt-manager.org/)
.

### VM setup

`virt-builder` is able to download the installer and create the disk image
with Fedora 33 or other distros.
It is also able to set the root password and inject the ssh public key,
simplifying the creation of guest disk image:

```shell
VM_IMAGE="vsockguest_f33.qcow2"

host$ virt-builder --root-password=password:mypassword \
        --ssh-inject root:file:/home/user/.ssh/id_rsa.pub \
        --output=${VM_IMAGE} \
        --format=qcow2 --size 10G --selinux-relabel \
        --update fedora-33
```

Once the disk image is ready, we create our VM with `virt-install`.
We can specify the VM settings like the number of vCPUs, the amount of RAM,
and the `CID` assigned to the VM [42]:

```shell
host$ virt-install --name vsockguest \
        --ram 2048 --vcpus 2 --os-variant fedora33 \
        --import --disk path=${VM_IMAGE},bus=virtio \
        --graphics none --vsock cid.address=42
```

After the creation of the VM, we will remain attached to the console and 
we can detach from it by pressing `ctrl-]`.

We can reattach to the console in this way:

```shell
host$ virsh console vsockguest
```

If the VM is turned off, we can boot it and attach directly to the console
in this way:

```shell
host$ virsh start --console vsockguest
```

### ncat like

It's possible to use `socat` like `ncat`, transferring `stdin` and `stdout` via VSOCK.

#### Guest listening

In this example we start `socat` in the guest listening on port `1234`:

```shell
guest$ socat - VSOCK-LISTEN:1234
```

Then we connect from the host using the `CID 42` assigned to the VM:

```shell
host$ socat - VSOCK-CONNECT:42:1234
```

At this point we can exchange characters between guest and host, since `stdin`
and `stdout` are linked through the VSOCK socket.


#### Host listening

In this example we do the opposite, starting `socat` in the host listening
on port `1234`:

```shell
host$ socat - VSOCK-LISTEN:1234
```

Then, in the guest, we connect to the host using the *well defined* `CID 2`.
It's always used to reach the host:

```shell
guest$ socat - VSOCK-CONNECT:2:1234
```

### ssh over VSOCK

The coolest feature of `socat` is to concatenate sockets of different address 
families, so in this example we redirect ssh traffic through VSOCK socket
exposed by the VM.

This example could be useful if the VM doesn't have any NIC attached and
we want to provide some network connectivity, like the ssh access.

First of all, in the guest we start `socat` linking the VSOCK socket listening on
port 22, to a TCP socket which will connect to the local TCP port 22 where the
ssh server is listening:

```shell
guest$ socat VSOCK-LISTEN:22,reuseaddr,fork TCP:localhost:22
```

On the host we link a TCP socket listening on a port of our choice (e.g. 4321)
to the guest port 22 just opened using VSOCK:

```shell
host$ socat TCP4-LISTEN:4321,reuseaddr,fork VSOCK-CONNECT:42:22
```

Finally from the host we can connect to the guest using ssh on the local port
4321, where `socat` is listening:

```shell
host$ ssh -p 4321 root@localhost
```

`socat` redirects all the traffic between the sockets and allow us to use ssh over
VSOCK to reach the guest.

### Connecting sibling VMs

Another scenario where socat can be very useful is connecting two sibling VMs
running on the same host.

Currently this is not supported by `vhost-vsock`, so we can use `socat` to
concatenate the two VMs.

Let's see an example: suppose we launch another VM with `CID = 24` on
the same host. Now we want to use `ncat` in the VMs to communicate with each
other.

Guest 42 will listen on port 1234 and guest 24 will initiate the connection.
But before we do this we need to set up the bridge in the host with `socat` to
allow the two VMs to communicate:

```shell
host$ socat VSOCK-LISTEN:1234 VSOCK-CONNECT:42:1234
```

At this point we can launch `ncat` in the VMs and communicate:

```shell
guest_42$ nc --vsock -l 1234
```

Note: as destination CID we have to use the host's well-known CID (2) because
we neet to connect to the `socat` bridge running in the host, that will
redirect packets correctly between the two machines:

```shell
guest_24$ nc --vsock 2 1234
```

