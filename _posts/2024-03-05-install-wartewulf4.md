---
author: Christian Goll
date: 2024-03-26 6:00:00+01:00
layout: post
license: CC-BY-SA-3.0
title: Installation guide for warewulf4
tags:
- Warewulf
- Warewulf4
---
# Warewulf
## Preface

In High Performance Computing (HPC) computing tasks are usually distributed
among many compute threads which are spread across multiples cores, sockets
and machines. These threads are tightly coupled together. Therefore compute
clusters consist of a number of largely identical machines that need to be
managed to maintain a well defined and identical setup across all nodes.
Once clusters scale up, there are many scalability factors to overcome.
Warewulf is there to address this 'administrative scaling'.

Warewulf is an operating system agnostic installation and management system
for HPC clusters.  
It is quick and easy to learn and use as many settings are pre-configured
to sensible defaults. It still provides the flexibility allowing to finely
tune the configuration to local needs.
It is released under the BSD license. Its source code is available at
https://github.com/warewulf/warewulf. This is where the development happens
as well.

## Installing Warewulf

Compute clusters consist of at least one management or head node which
is usually multi-homed connecting both to an external network and to a cluster
private network as well as multiple compute nodes which reside solely on the
private network. Other private networks dedicated to high speed task like RDMA
and storage access may exist as well.
Warewulf gets installed on one of the management nodes to manage and oversee
the installation and management of the compute nodes.
To install Warewulf on a cluster which is running openSUSE Leap 15.5 or
openSUSE Tumbleweed, simpy run:
```
zypper install warewul4
```
This package seamlessly integrates into a SUSE system and should therefore
be preferred over packages provided on Github.

During the installation the actual network configuration is written to
`/etc/warewulf/warewulf.conf`. These settings should be verified, as for
multi homed hosts a sensible pre-configuration is not always possible.

Check `/etc/warewulf/warewulf.conf` for the following values:
```
ipaddr: 172.16.16.250
netmask: 255.255.255.0
network: 172.16.16.0
```
where `ipaddr` should be the IP address of this management host.
Also check the values of `netmask` and `network` - these should match
this network.

Additionally, you may want to configure the IP addresse range for
dynamic/unknown hosts:
```
dhcp:
  range start: 172.16.26.21
  range end: 172.16.26.50
```

If the ISC dhcpd server is used (default on SUSE) make sure the value of
`DHCPD_INTERFACE` in the file `/etc/sysconfig/dhcpd`  has been set to the
correct value.

You are now ready to start the `warewulfd` service itself which delivers the
images to the nodes:
```
systemctl enable --now warewulfd.service
```

Now `wwctl` can be used to configure the the remaining services needed by
Warewulf. Run
```
wwctl configure --all
```
which will configure all Warewulf related services.

## Adding nodes and profiles to Warewulf

Warewulf uses the concept of profiles which hold the generalized settings
of the individual nodes. It comes with a predefined profile `default`, to
which all new node will be assigned, if not set otherwise. You may obtain
the values of the default profile with:
```
wwctl profile list default
```

Now, a node can be added with the command assigning it an IP address:
```
wwctl node add node01 -I 172.16.16.101
```
the MAC address is known for this node, you can specify this as well:
```
wwctl node add node01 -I 172.16.16.101 -H cc:aa:ff:ff:ee
```

For adding several nodes at once you may also use a node range which results
e.g.
```
wwctl node add node[01-10] -I 172.16.16.101
```
this will add the nodes with ip addresses starting at the specified address
and incremented by Warewulf.

## Importing a container

Warewulf uses a special [^footnote1] container as the base system to build OS
images for the compute nodes. This is self contained and independent of the
operating system installed on the Warewulf host.

To import an openSUSE Leap 15.5 container use the command
```
wwctl container import docker://registry.opensuse.org/science/warewulf/leap-15.5/containers/kernel:latest leap15.5 --setdefault
```
This will import the specified container for the default profile.

### Alternative container sources

Alternative containers are available from the openSUSE registry under the
science project at

https://registry.opensuse.org/cgi-bin/cooverview?srch_term=project%3D%5Escience%3A

or from the upstream Warewulf community repository

https://github.com/orgs/warewulf/packages?repo_name=warewulf-node-images

It is also possible to import an image from a `chroot` by using the path to
`chroot` as argument for `wwctl import`.

# Booting nodes

As a final preparation you should rebuild the container image by running
```
wwctl container build leap15.5
```
as well as all the configuration overlays with the command
```
wwctl overlay build
```
just in case the build of the image may have failed earlier due to an error.
If you didn't assign a hardware address to a node before you should set the
node into the discoverable state before powering it on. This is done with
```
wwctl node set node01 --discoverable
```

Now the node(s) can be powered on and will boot into assigned container.

To convenient log into compute nodes, you should now log out of and back
into the Warewulf host, as this way an ssh key will be created on the
Warewulf host which allows password-less login to the compute nodes.
Note, that this key is not pass-phrase protected. If you require to protect
your private key by a pass phrase, it is probably a good idea to do so now:
```
ssh-keygen -p -f $HOME/.ssh/cluster
```
Also you should run
```
wwctl configure hostlist
```
to add the new nodes to the file `/etc/hosts`.

# Additional configuration

The configuration files for the nodes are managed as Golang text templates.
The resulting files are overlayed over the node images. There are two ways
of transport for the overlays to the compute node, the
* system overlay
* runtime overlay
where the system overlay is baked into the boot image of the compute node
while the runtime overlay is updated on the nodes on a regular base
(1 minute per default) via the `wwclient` service.

In the default configuration the overlay called `wwinit` is used as system
overlay. You may list the files in this overlays with the command:
```
wwctl overlay list wwinit -a
```
which will show a list of all the files in the overlays. Files ending with the
suffix `.ww` are interpreted as template by Warewulf and the suffix is removed
in the rendered overlay.
The content of the overlay can be shown using the command
```
wwctl overlay show wwinit /etc/issue.ww
```
To render the template  using the values for node01 use:
```
wwctl overlay show wwinit /etc/issue.ww -r node01
```
The overlay template itself may be edited using the command
```
wwctl overlay edit wwinit /etc/issue.ww
```

Please note that after editing templates the overlays aren't updated
automatically and should be rebuild with the command
```
wwctl overlay build
```

The variables available in a template can be listed with
```
wwctl overlay show debug /warewulf/template-variables.md.ww
```

# Modifying the container

The node container is a self contained operating system image. You can open a
shell in the image with the command
```
wwctl container shell leap15.5
```
After you have opend a shell you may install additional software using
`zypper`.

The shell command provides the option `--bind` which allows to mount arbitrary
host directories into the container during the shell session.

Please note that if a command exits with a status other than zero the image
won't be rebuilt automatically. Therefore it is advised to rebuild the
container with
```
wwctl conainer build leap15.5
```
after any change.


# Network configuration

Warewulf allows to configure multiple network interfaces for the compute nodes.
You can add another network interface for example for infiniband using the
command
```
wwctl node set node01 --netname infininet -I 172.16.17.101 --netdev ib0 --mtu 9000 --type infiniband
```
This will add the infiniband interface `ib0` to the node `node01`. You can now
list the network interfaces of the node:
```
wwctl node list -n
```
As changes in the settings are not propagated to all configuration files, the
node overlays should be rebuilt after this change running the command:
```
wwctl overlay build
```
After a reboot these changes will be present on the nodes, in the above case
the Infiniband interface will be active on the node.

A more elegant way to get same result is to create a profile to hold the values
which are the same for all interfaces. In this case these are `mtu` and the
`netdev`.
A new profile for an Infiniband network is created using the command
```
wwctl profile add infiniband-nodes --netname infininet --netdev ib0 --mtu 9000 --type infiniband
```
Once this has been created, you may add this profile to a node and remove the
node specific settings which are now part of the common profile by executing:
```
wwctl node set node01 --netname infininet --netdev UNDEF --mtu UNDEF --type UNDEF --profiles default,infiniband-nodes
```
You may list the data in a profile using this command:
```
wwctl profile list -A infiniband-nodes
```

# Secure Boot

## Switch to grub boot

Per default Warewulf boots nodes via iPXE, which isn't signed by SUSE and
can't be used when secure boot is enabled. In order to switch to grub as boot
method you will have add/change following value in `/etc/warewulf/warewulf.conf`
```
warewulf:
  grubboot: true
```
After this change you will have to reconfigure `dhcpd` and `tftp` executing
```
wwctl configure dhcp
wwctl configure tftp
```
and rebuild the overlays with the command
```
wwctl overlay build
```
Also make sure that the packages `shim` and `grub2-x86_64-efi` for x86-64
or `grub2-arm64-efi` for arm are installed in the container. `shim` is
required by secure boot.

## Cross product secure boot

If secure boot is enabled on the compute nodes and you want to boot different
products make sure that the compute nodes boot with the so called 'http' boot
method: For secure boot the signed `shim` needs to match the signature of the
other pieces of the boot chain - including the kernel. The 'http' method is
handled by `warewulfd` which will look up the image to boot and pick the shim
from the image to deploy to this node. Otherwise, the initial `shim` for PXE
boot, which is the default boot method, is extracted from the host running the
warewulfd server. Make sure, the node container contains the `shim` package.
The host system `shim` will also be used for nodes which are in `discoverable`
state and subsequently have no hardware address assigned yet.

# Disk management

It is possible to manage the disks of the compute nodes with Warewulf. Here,
Warewulf itself doesn't manage the disks, but creates a configuration and
service files for `ignition` to do this job.

## Prepare container

As `ignition` and its dependencies aren't installed in most of the containers
you should install the packages `ignition` and `gptfdisk` in the container

```
wwctl container exec <container_name> zypper -n in -y ignition gptdisk
```

## Add disk to configuration

For storage devices all the necessary structures must be configured which are

* physical storage device(s) to be used
* partition(s) on the disks
* filesystem(s) to be used

### Disks

The path to the device e.g. `/dev/sda` must be used As `diskname`.
The only valid configuration for disks is `diskwipe` which should be
self explanatory.

### Partitions

The `partname` is the name to the partition whick iginition uses as the path
for the device files, e.g. `/dev/disk/by-partlabel/$PARTNAME`.

Additionally the size and number of the partition need be specified for all
but the last partition (the one with the highest number) in which case this
partition will be extended to the maximal size possible.

You should also set the boolean variable `--partcreate` so that a parition
is created if it doesn't exist.

### Filesystems

Filesystems are defined by the partition which contains them, so the
name should have the format `/dev/disk/by-partlabel/$PARTNAME`. A filesystem
needs to have a path if it is to be mounted, but its not mandatory.

Ignition will fail, if there is no filesystem type defined.

## Examples

You can add a scratch partition with
```
wwctl node set node01 \
  --diskname /dev/vda --diskwipe \
  --partname scratch --partcreate \
  --fsname scratch --fsformat btrfs --fspath /scratch --fswipe
```
This will be the only (and last) partition, therefore it does not
require a size.
To add another partition as swap partition, you many run:
```
wwctl node set n01 \
  --diskname /dev/vda \
  --partname swap --partsize=1024 --partnumber 1 \
  --fsname swap --fsformat swap --fspath swap
```
This adds the partition number 1 which will be placed before
the `scratch` partition.


[^footnote1]: This container is special only in that it is bootable, ie it
    contains a kernel and an init-implementation (systemd).
