---
author: Christian Goll
date: 2023-08-15 12:00:00+01:00
layout: post
license: CC-BY-SA-3.0
title: 
tags:
- kvm
- qemu
- ipmi
---

# Virtual ipmi interface

Kvm/qemu support virtual ipmi devices which can be used to test ipmitool related
commands on a virtual machine. These feature is barely documented, but is use able.

There are two components, the ipmi device of kvm/qemu and a ipmi simulator which is 
part of the `ipmitools` package. 

## Adding the ipmi device

The ipmi device can be added to qemu as command line argument, which is
```
-device ipmi-bmc-sim,id=virt-bmc -device pci-ipmi-kcs,bmc=virt-bmc,id=virt-bmc-pci
```

** Warning **

This changes the pci is, so other device need new ids


Unfortunately this device isn't presented by the libvirt interfaces, but can be added 
to the `xml` configuration directly with the following lines
```
  <qemu:commandline>
    <qemu:arg value="-device"/>
    <qemu:arg value="ipmi-bmc-sim,id=virt-bmc"/>
    <qemu:arg value="-device"/>
    <qemu:arg value="pci-ipmi-kcs,bmc=virt-bmc,id=virt-bmc-pci"/>
  </qemu:commandline>
```


## Connect device to simulator

Its also possible to connect this kvm/qemu to a ipmi simulator running on the 
host. The xml changes to
```
  <qemu:commandline>
    <qemu:arg value="-chardev"/>
    <qemu:arg value="socket,id=ipmi0,host=localhost,port=9002,reconnect=10"/>
    <qemu:arg value="-device"/>
    <qemu:arg value="ipmi-bmc-extern,id=virt-bmc,chardev=ipmi0"/>
    <qemu:arg value="-device"/>
    <qemu:arg value="pci-ipmi-kcs,bmc=virt-bmc,id=virt-bmc-pci"/>
  </qemu:commandline>
```
where we had to add an additonal chardev which connects to the ipmi simulator.
The simulator can be started with
```
ipmi_sim /etc/ipmi/lan.conf -f /etc/ipmi/ipmisim1.emu -s $IPMISTATDIR
```

The configuration files under `/etc/ipmi` are part of the ipmitool package.
Important is the `$IPMISTATDIR` which can contain addtional SDRs. A simple SDR
with a temperature sensor can be added adding the followiing lines to the file
`$IPMISTATDIR/ipmisim1/sdr.20.main`:
```
last_add_time:i:1691752652
6:d:\06\00Q\11\140\03\80\00\00\10\00\08\02\00\c9mm2frudev
5:d:\05\00Q\02" \00\02\08\01\00\00%o\03\00\03\00\03\00\c0\00\00\00\00\00\00\00\00\00\00\c7mm1pres
4:d:\04\00Q\0120\00\01\07\01E\00\01\01\00\00\00\00\00\00\00\01\00\00\01\00\00\01\01\00\00\00\00\00\ff\00\00\00\00\00\00\00\00\00\00\00\00\c7SubTemp
3:d:\03\00Q\011 \00\01\07\01E\00\01\01\00\00\00\00\00\00\00\01\00\00\01\00\00\01\01\00\00\00\00\00\ff\00\00\00\00\00\00\00\00\00\00\00\00\c6MBTemp
2:d:\02\00Q\03\14 \00\00\07\01#o\00\00\00\00\c8watchdog
1:d:\01\00Q\12\14 \00\00\8f\00\00\00\07\01\00\c9IPMI sim1
```

Have a lot of fun.
