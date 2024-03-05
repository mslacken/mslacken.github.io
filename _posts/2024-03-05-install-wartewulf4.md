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

# Installing warewulf4

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
