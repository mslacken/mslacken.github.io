---
author: Christian Goll
date: 2023-08-15 12:00:00+01:00
layout: post
license: CC-BY-SA-3.0
title: 
tags:
- confluent
- xcat
---

# Confluent/Xcat quickstart guide for (open)SUSE

At first you need a least a compute cluster containing of one orchestrator and one node. 
For testing purpose a terrform configuration for setting this up is available under:

https://github.com/warewulf/warewulf-testenv

## Install confluent

Add the reop and the keys
```
rpm --import https://hpc.lenovo.com/yum/latest/suse15/x86_64/lenovohpckey.pub
zypper install https://hpc.lenovo.com/yum/latest/suse15/x86_64/lenovo-hpc-zypper-1-1.x86_64.rpm
```
After that the confluent package can be installed with
```
zypper install lenovo-confluent

```

Unfortunately the dependency the python dbm package is missing and must be installed with
```
zypper install python3-dbm
```

To enable the service run

```
systemctl enable confluent --now
```
and enable also tftp with
```
systemctl enable --now tftp

```

## Enable web client

Follow the steps under https://hpc.lenovo.com/users/documentation/installconfluent_suse.html
 or do following

```
cp /etc/apache2/vhosts.d/vhost-ssl.template /etc/apache2/vhosts.d/mySSL.conf
```
and create the SSL certificate with
```
osdeploy initialize -t
```
and enable SSL for apache with
```
a2enmod rewrite
a2enflag SSL
systemctl enable apache2 --now
```

Now a user `root` in this case can be added to the web gui with

```
confetty create /users/root role=admin
```

# Configure cluster

## Add nodes

Add global variables to `everything` group in which all nodes are part of
```
nodegroupattrib everything deployment.useinsecureprotocols=firmware console.method=ipmi dns.servers=172.16.16.1 dns.domain=cluster.net net.ipv4_gateway=172.16.16.1
```

The option `deployment.useinsecureprotocols=firmware` allows iPXE installations `deployment.useinsecureprotocols=firmware` allows iPXE installations. Secrets and passwords can be added with
```
nodegroupattrib everything -p bmcuser bmcpass crypted.rootpassword crypted.grubpassword
```
which will add the BMC/User with password and the cluster wide root password and as well the password to access grub.

Now the nodes can be added with

```
for i in {1..4}; do nodename=n$(printf %02i $i); nodedefine $nodename net.ipv4_address=172.16.16.${i}; done
```

Add the entries to `/etc/hosts` with

```
noderun -n n01-n04 echo {node}  {net.ipv4_address} >> /etc/hosts
```

## Add OS

Before any OS can be added certificates for the OS deployment must be create with

```
osdeploy initialize -i
```
and import the SLE iso with 
```
osdeploy import SLE-15-SP5-Full-x86_64-GM-Media1.iso
```
the imported image can be checked with
```
osdeploy list

```


