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
. Warewulf can work without any of these services but this requires additional configuration steps.
Now its necessary to set the right values for `/etc/warewulf/warewulf.conf`, which are in most cases the  IP address of the master node. For the ipaddr in the line
```
ipaddr: 10.10.93.254
```
has to be changed to the IP address of your warewulf server. In the section `dhcp` the values for
```
  range start: 10.10.93.10
  range end: 10.10.93.250
```
also have to be change accordingly.
At this step you may also want to configure the nfs shares, which are exported from the master node. The shares are configured in the section `nfs`. If you want to export `/srv/software/spack` to your compute nodes, you have to add
```
    - path: /srv/software/spack
      export options: ro,sync,no_root_squash,no_subtree_check
      mount: true
```
Before starting all the necessary services, the configuration of the `dhcpd` service must be altered. For this add the name of the network interface to `DHCPD_INTERFACE` in the file `/etc/sysconfig/dhdpd`.
After this steps, you can now let configure warewulf the rest of the system with command
```
wwctl configure -a
```
Now the systemd service `warewulfd` should be started with the command
```
systemctl enable --now warewulfd
```
The `warewulfd` service delivers the system images aka containers and the overlays to the compute nodes.

# Get the image for the compute nodes
The easiest way to get the image for the compute is to download it directly from the SUSE registry with the command
```
wwctl container import docker://registry.suse.com/sle15-sp4-hpc/hpc-ww4-kernel sle15sp4
```
what will import the image of SLE 15SP4 HPC as `sle15sp4` to ww4. The image can now be registered with the command
```
SUSEConnect --root $(wwctl container show sle15sp4) -e YOU@EXAMPLE.COM -r YOUR-REGISTRATION-KEY
```
As kernel and base image are handled separately in ww4, the kernel with its modules has to installed with the command
```
wwctl kernel import -D -C sle15sp4
```
In order to finish the basic configuration you will have to modify the values for the
* kernel
* container
* netmaks
* gateway
in the `default` profile with the command
```
wwctl profile set default --container=sle15sp4 --kernel=sle15sp4 --netname=default --netmask=255.255.255.0 --gateway=10.10.93.1
```
You may want to omit the `--gateway` if the nodes should not be able to connect to external networks.
# Add nodes to ww4

You can now add a node with the command
```
wwctl node add node01 --netname default --netdev lan0 -I 10.10.93.11
```
In the final step the overlays for the nodes have to build with the command
```
wwctl overlay build
```
Now you can turn on the node and watch it boot.

