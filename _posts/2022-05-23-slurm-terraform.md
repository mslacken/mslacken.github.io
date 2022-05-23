---
author: Christian Goll
date: 2022-05-23 12:00:00+01:00
layout: post
license: CC-BY-SA-3.0
title: 
categories:
- terraform
- kiwi
- slurm
- Leap
- Tumbleweed
tags:
- terraform
- kiwi
- slurm
- Leap
- Tumbleweed
---
# Terraform and kiwi
Setting up a slurm cluster for testing purpose is always time consuming and error prone. Especially if just some smaller changes in the configuration have to be tested. 

In order to automate this, I have written a small [test setup](https://github.com/mslacken/terraform-slurm) based on [kiwi](https://github.com/OSInside/kiwi) and [terraform](https://github.com/hashicorp/terraform). 

The kiwi part builds one image and bakes in the `slurm.conf` and a proper shared nfs `/home`. As all nodes boot from the same image the munge key, which is generated at install time, is the same.

So all configuration files are in the right place.

The network configuration is managed with the terraform configuration and with `DHCLIENT_SET_HOSTNAME="yes"` in the file `/etc/sysconfig/network/dhcp` the dhpd name is the FQDN.

# Usage
The terraform providers have to be installed with
```
sudo terraform init
```
Now you can build a image with
```
./build-image.sh leap15.4
```

With the image the cluster can be started with 
```
sudo terraform apply -var="image=/var/tmp/leap15.4-current/Leap-15.4_appliance.x86_64-1.15.3.qcow2"
```

Quiet easy?

# Customization
The individual configurations for the images are in their directories. E.g. the configuration for the openSUSE Leap 15.4 image is the file `leap15.4/config.xml`.

The configuration of the services comes from the files in the `assets/` directory, but as the distribution directory is copied over this directory during the image build process, e.g. a distribution specific `slurm.conf` would resided in `tw/root/etc/slurm/slurm.conf`.

