---
author: Christian Goll
date: 2024-03-05 12:00:00+01:00
layout: post
license: CC-BY-SA-3.0
title: 
tags:
- warewulf
- warewulf4
---
# Warewulf
## Preface

Warewulf is operating system agnostic installation and management system for HPC
cluster. Although many things are pre configured for a fast start up it sill has
the possibilty for fain grained configurations.
It release under a BSD license and its source are available under
https://github.com/warewulf/warewulf
where also the development happens.

## Installing warewulf4

Warewulf can be installed with the command
```
zypper install warewul4
```
which install the warewulf package build by SUSE on your host. This package has
some minor SUSE specific modification and should be preferred over the package
provided on the github page.

During the installation the actual network configuration is written to the
configuration file `/etc/warewulf/warewulf.conf`, but should be checked in any
case, as for multi homed hosts a sensible configuration is not possible.

Following values should be checked in the file `/etc/warewulf/warewulf.conf`:
```
ipaddr: 172.16.16.250
netmask: 255.255.255.0
network: 172.16.16.0
``` 
where the value of `ipaddr` must be the ip address for the host running the
`warwulfd` service and the values `netmask` and `network` are the network and
its associated ip address.

Additionally the range of ip addresses for dynamic/unknown hosts should be
configured under 
```
dhcp:
  range start: 172.16.26.21
  range end: 172.16.26.50
```

If the isc dhcpd server is used in the file `/etc/sysconfig/dhcpd` the value of
`DHCPD_INTERFACE` has to be set to right value.

You are now ready to start the `warewulfd` service itself which delivers the
images to the nodes.

Now `wwctl` can be used to configure the rest of the services needed by
warewulf. Use 
```
wwctl configure --all
```
which will configure all warewulf related services.

## Adding nodes and profiles to warewulf

Warewulf uses the concept of profiles which hold the generalized settings
of the individual nodes. It comes with the predefined profile `default`, to
which will be assigned to every new node, if not set otherwise. The values of
the default profile can be listed with 
```
wwctl profile list default
```

Now a node can be added with the command
```
wwctl node add node01 -I 172.16.16.101
```
and if the mac address is known for this node, you can use
```
wwctl node add node01 -I 172.16.16.101 -H cc:aa:ff:ff:ee
```

For adding several nodes at once you can also use a node range which results
e.g.
```
wwctl node add node[01-10] -I 172.16.16.101
```
what will add the nodes and the ip address is then also incremented by warewulf.

## Importing a container

A container is the operating system for the compute nodes. It is self contained
and is independent of the operating system installed on the warewulf host. Only
the registered extensions and modules with SUSEConnect registered on the host
are also available in the container.

To import a openSUSE Leap 15.5 container use the command
```
wwctl container import docker://registry.opensuse.org/science/warewulf/leap-15.5/containers/kernel:latest leap15.5 --setdefault
```
what will import an openSUSE Leap 15.5 and make it the container of the default
profile.

### Alternative container sources

Alternative container can be obtained form the openSUSE registry under the
science project under following url

https://registry.opensuse.org/cgi-bin/cooverview?srch_term=project%3D%5Escience%3A

or from the upstream warewulf community repository under

https://github.com/orgs/warewulf/packages?repo_name=warewulf-node-images

It is also possible to import an image from a `chroot` by using the path to
`chroot` as argument for `wwctl import`.

# Booting nodes

As a final preparation you should rebuild the container image with the command
```
wwctl container build leap15.5
```
and the all the configuration overlays with the command
```
wwctl overlay build
```
because the build of the image may have failed due to an error earlier.
If you didn't assign a hardware address to a node before you should set the node
into the discoverable state before powering it on. This is done with
```
wwctl node set node01 --discoverable
```

Not the node(s) can be powered on and will boot into assigned container.

For a convenient experience you should now logout and login into the warewulf
host, as this way a ssh key for password less login is created on the warewulf
host. Also you should run 
```
wwctl configure hostlist
```
to add the new nodes to the file `/etc/hosts`.

# Additional configuration

The configuration files for the nodes are managed with golang text templates.
There are two ways of transport for the overlays to the compute node, the
* system overlay
* runtime overlay
where the system overlays is baked into the boot image of the compute node and
runtime is updated on a regular base (1 minute per default) via the `wwclient`
service.

In the default configuration the overlay called `wwinit` is used as system
overlay. You can list the files in this overlays with the command:
```
wwctl overlay list wwinit -a
```
what will give a list of all the files in the overlays. Files ending with the
suffix `.ww` are interpreted by warewulf and the suffix is removed in the
rendered overlay. 
The content of the overlay can be shown with the command
```
wwctl overlay show wwinit /etc/issue.ww
```
and with the command
```
wwctl overlay show wwinit /etc/issue.ww -r node01
```

The overlay itself can be edited with the command
```
wwctl overlay edit wwinit /etc/issue.ww
```

Please note that there after an edit the overlay aren't updated automatically
and the overlays should be rebuild with the command
```
wwctl overlay build
```

An overview of the variables available in the template can be listed with
```
wwctl overlay show debug /warewulf/template-variables.md.ww
```

# Modifying the container

A container is a self contained operating system image. You shell into the image
with the command
```
wwctl container shell leap15.5
```
what will open shell within in image. After you have shelled into the image
additional software can be installed with `zypper`. 

The shell command has also the `--bind` option which allows to mount arbitrary
directories from the host into the container during the shell session. 

Please note that if a command exits with a status other than zero, then the image
isn't rebuilt automatically. So its also advised to rebuild the container with
```
wwctl conainer build leap15.5
```
after any change.


# Network configuration

Multiple network interfaces for the compute nodes can be configure with warewulf. 
You can add another network interface with the command
```
wwctl node set node01 --netname infininet -I 172.16.17.101 --netdev ib0 --mtu 9000 --type infiniband
```
what will add the infiniband interface `ib0` to the node `node01`. You can now
list the network interfaces of the node with the command
```
wwctl node list -n
```

As changes in the settings are not tracked to all the settings, the overlays of
the nodes should be rebuilt after this change with the command
```
wwctl overlay build
```
After a reboot these changes will be present on the nodes, so the interface will
be active on the node.

A more elegant way to get same result is to create a profile to hold the values
which are the same for all interfaces which are in this case the `mtu` and the
`netdev`.
A new profile is created with the command
```
wwctl profile add infiniband-nodes --netname infininet --netdev ib0 --mtu 9000 --type infiniband
```
after that add this profile to the node and remove the node specific settings
which are now in profile from the node with the command
```
wwctl node set node01 --netname infininet --netdev UNDEF --mtu UNDEF --type UNDEF --profiles default,infiniband-nodes
```
which will remove the node specific settings of the network and set the
profiles.


# Switch to grub boot

Per default warewulf boots the node via iPXE, which isn't signed by SUSE and
can't be used for secure boot. In order to enable grub as boot method you will
have add/change following value in `/etc/warewulf/warewulf.conf`
```
warewulf:
  grubboot: true
```
After this change you will have to reconfigure `dhcpd` and `tftp` with commands
```
wwctl configure dhcp
wwctl configure tftp
```
and rebuild the overlays with the command
```
wwctl overlay build
```

Also make sure that the `shim` and `grub` are installed in the container. For
openSUSE/SUSE the package names are `shim` and `grub2-x86_64-efi`, as the `shim`
package is needed for secure boot.

## Cross distribution secure boot

If secure boot is enabled on the compute nodes and you want to boot different
distributions make sure that the compute nodes boot with the so called http boot
method. The reason for this, that the initial `shim` for PXE boot, which is
default boot method, is extracted from the host running the warewulfd server.
This `shim` is also used for nodes which are in `discoverable` state and
subsequently have no hardware address assigned yet.

# Disk management

It is possible to manage the disks of the compute nodes with warewulf. As
warewulf itself doesn't manage the disks, but creates a configuration and
service files for `ignition` to do this job.

As `ignition` and its dependencies aren't installed in most of the containers you should install following packages in the container

* `ignition`
* `gptfdisk`




