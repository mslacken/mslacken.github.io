---
author: Christian Goll
email: CGoll@suse.com
date: 2021-02-27 
layout: post
license: CC-BY-SA-3.0
title: Install a SLE15-SP3 HPC-cluster with WW4
categories:
- HPC
- ww4
tags:
- ww4
- warewulf
- SLE15-SP3
---
# Install warewulf on the master node
On the master node you have to install warewulf v4 (ww4) with the command
```
sudo zypper install warewulf4
```
this will install ww4 with its depencies, which are
* dhcpd server
* nfs-kernel server
* tfptp
. Warewulf can work without any of these services but this requires addtinal configuration steps.
Now its necessary to set the right values for `/etc/warewulf/warewulf.conf`, which are in most cases the following:
* IP address of the master node
* dynamic range for the dhpd server
At this step you may also wan't to configure the nfs shares, which are exported from the master node.
