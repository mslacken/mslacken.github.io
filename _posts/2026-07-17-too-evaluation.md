---
author: Christian Goll
date: 2026-07-17 12:00:00+01:00
layout: post
license: CC-BY-SA-3.0
title: How to Create an MCP Server
categories:
- programming
- mcp
tags:
- machine learning
- programming
- toon
---

# Evaluating toon in real world scenario

Token oriented notation abbreviated as toon is the idea to rewrite json in such way that less characters and thus less tokens are generated. The concept is more deeply explained at https://toonformat.dev/ and a simple example for this is following. When asking the systemd-mcp server for all inactive services on a system you will get without toon, following output:
```

{
  "users": [
    { "id": 1, "name": "Alice", "role": "admin" },
    { "id": 2, "name": "Bob", "role": "user" }
  ]
}
```

With 119 charcates reduces to 53 characters as in toon it would be 
```
users[2]{id,name,role}:
  1,Alice,admin
  2,Bob,user
```
see also the systemd example at the end of this text.
As token savings are an easy way to bring AI costs down , increases response times and allows a better use of the context window this seems to be an easy win. Still changing the fundamental response from the MCP server seems may also have down sides. So I decided to make some real world experiments and see if there are any down sides of this, were the most severe would incorrect responses from the LLM or more tool calls nullifying the token savings.

## MCP server and the implementation

As I was working on the systemd-mcp server https://github.com/openSUSE/systemd-mcp I decided to use this project as base.
The standard GO implementation of MPC at https://github.com/modelcontextprotocol/go-sdk doesn't offer toon yet, I used https://github.com/aj-geddes/toon-context-mcp . The implementation is relatively straight forward as the real payload of a MCP is text field which contains the as json marshalled internal structure. So this text field must now marshalled using toon. The outer json structrue which is used for communcation bewtween the MCP server and the client is left intact.

## Test bed

Creating the test bed is a bit more complicated and needs some refining. At first the MCP server needs a real, life running systemd process to connect to. So a reproducible and secure solution for this is to use virtual machine. The tofu (OSS terraform) definition which can be found at https://github.com/mslacken/EvalToon/blob/main/tofu/leap-cloud.tf which uses an openSUSE Tumbleweed cloud image as it can be configured via cloud-init.
A note able side quest here was to get the systemd-mcp server into the VM, without compliling it the whole would consume some CPU cycles and using the rpm package won't work as I didn't push the toon changes upstream, yet. The canconical way to copy now a binary to a VM would be use simply copy the file VM with `scp`. I didn't want to use this method as the ssh server was disabled and fixing this, I wanted to have as task to the LLM. This leaves following possibilities (or some more):
* use 9p or similar FS
* use a readonly FS containing only the binary
As the 9p solution would require abolute paths in the cloud init config, I just created a second virtual disk which just contains the binary. After the start of the VM I could just `dd` to dump the binary into the FS. gg



## systemd example
```
state: inactive

{
state:

"inactive"

units:[
0: "systemd-battery-check.service"
1: "emergency.service"
2: "initrd.target"
3: "dm-event.service"
4: "check-battery.service"
5: "apt-daily.service"
6: "plymouth-start.service"
7: "rc-local.service"
8: "boot.automount"
9: "ip6tables.service"
10: "boot.mount"
11: "blockdev@dev-disk-by\x2duuid-41b87736\x2d08cc\x2d6bd0\x2d85de\x2dd844cd161676.target"
12: "systemd-soft-reboot.service"
13: "firewalld.service"
14: "plymouth-quit.service"
15: "blockdev@dev-vda2.target"
16: "remote-cryptsetup.target"
17: "dracut-pre-pivot.service"
18: "modprobe@fuse.service"
19: "modprobe@efi_pstore.service"
20: "dracut-cmdline.service"
21: "lvm2-lvmpolld.service"
22: "dracut-pre-mount.service"
23: "initrd-root-device.target"
24: "YaST2-Second-Stage.service"
25: "systemd-repart.service"
26: "cryptsetup-pre.target"
27: "emergency.target"
28: "umount.target"
29: "systemd-hostnamed.service"
30: "nss-lookup.target"
31: "dracut-initqueue.service"
32: "initrd-parse-etc.service"
33: "shutdown.target"
34: "soft-reboot.target"
35: "initrd-switch-root.service"
36: "issue-generator.service"
37: "iptables.service"
38: "veritysetup-pre.target"
39: "backup-sysconfig.service"
40: "dev-ttyAMA0.device"
41: "initrd-usr-fs.target"
42: "systemd-modules-load.service"
43: "initrd-root-fs.target"
44: "systemd-tpm2-setup-early.service"
45: "final.target"
46: "systemd-hibernate-resume.service"
47: "audit-rules.service"
48: "xdm.service"
49: "nss-user-lookup.target"
50: "initrd-switch-root.target"
51: "home.mount"
52: "poweroff.target"
53: "systemd-firstboot.service"
54: "blockdev@dev-disk-by\x2duuid-BE2D\x2dB7C2.target"
55: "remote-fs-pre.target"
56: "ntpd.service"
57: "systemd-rfkill.service"
58: "logrotate.service"
59: "systemd-udev-settle.service"
60: "ebtables.service"
61: "systemd-tmpfiles-clean.service"
62: "serial-getty@ttyS2.service"
63: "getty-pre.target"
64: "remote-veritysetup.target"
65: "apache2.target"
66: "apparmor.service"
67: "proc-sys-fs-binfmt_misc.mount"
68: "sysroot.mount"
69: "modprobe@configfs.service"
70: "ca-certificates.service"
71: "systemd-poweroff.service"
72: "rescue.target"
73: "systemd-journald-audit.socket"
74: "nftables.service"
75: "rescue.service"
76: "fstrim.service"
77: "systemd-ask-password-console.service"
78: "systemd-hibernate-clear.service"
79: "dracut-mount.service"
80: "systemd-timesyncd.service"
81: "modprobe@drm.service"
82: "wicked.service"
83: "systemd-pstore.service"
84: "systemd-binfmt.service"
85: "initrd-cleanup.service"
86: "sshd.service"
87: "syslog.service"
88: "display-manager.service"
89: "systemd-quotacheck-root.service"
90: "initrd-udevadm-cleanup-db.service"
91: "boot-sysctl.service"
92: "systemd-networkd-wait-online.service"
93: "hv_kvp_daemon.service"
94: "initrd-fs.target"
95: "serial-getty@ttyS1.service"
96: "ipset.service"
97: "dracut-pre-udev.service"
98: "backup-rpmdb.service"
99: "dracut-pre-trigger.service"
100: "dracut-shutdown-onfailure.service"
101: "serial-getty@ttyAMA0.service"
102: "wtmpdb-rotate.service"
103: "syslog.socket"
104: "plymouth-quit-wait.service"
105: "sshd-keygen.service"
106: "wtmpdbd.service"
]
}
```


```
units[107]: systemd-repart.service,ntpd.service,boot-sysctl.service,systemd-journald-audit.socket,initrd-fs.target,systemd-tpm2-setup-early.service,apparmor.service,systemd-pstore.service,backup-sysconfig.service,iptables.service,systemd-modules-load.service,modprobe@drm.service,dev-ttyAMA0.device,modprobe@efi_pstore.service,issue-generator.service,nss-lookup.target,syslog.service,getty-pre.target,final.target,initrd-usr-fs.target,systemd-hibernate-resume.service,systemd-ask-password-console.service,syslog.socket,sshd.service,modprobe@configfs.service,poweroff.target,ebtables.service,rescue.target,emergency.target,sysroot.mount,ip6tables.service,rc-local.service,blockdev@dev-vda2.target,systemd-rfkill.service,apt-daily.service,dracut-initqueue.service,serial-getty@ttyAMA0.service,check-battery.service,systemd-timesyncd.service,soft-reboot.target,apache2.target,home.mount,systemd-poweroff.service,nftables.service,lvm2-lvmpolld.service,dracut-shutdown-onfailure.service,plymouth-quit-wait.service,systemd-udev-settle.service,serial-getty@ttyS2.service,systemd-hibernate-clear.service,boot.automount,YaST2-Second-Stage.service,plymouth-start.service,initrd-udevadm-cleanup-db.service,dracut-pre-mount.service,nss-user-lookup.target,serial-getty@ttyS1.service,wicked.service,systemd-battery-check.service,initrd-parse-etc.service,emergency.service,hv_kvp_daemon.service,ca-certificates.service,systemd-soft-reboot.service,backup-rpmdb.service,modprobe@fuse.service,dracut-pre-trigger.service,dm-event.service,plymouth-quit.service,"blockdev@dev-disk-by\\x2duuid-BE2D\\x2dB7C2.target",audit-rules.service,rescue.service,initrd-root-device.target,systemd-firstboot.service,"blockdev@dev-disk-by\\x2duuid-41b87736\\x2d08cc\\x2d6bd0\\x2d85de\\x2dd844cd161676.target",shutdown.target,remote-veritysetup.target,initrd-root-fs.target,initrd.target,display-manager.service,wtmpdb-rotate.service,systemd-hostnamed.service,remote-cryptsetup.target,dracut-pre-udev.service,systemd-quotacheck-root.service,initrd-switch-root.target,dracut-pre-pivot.service,proc-sys-fs-binfmt_misc.mount,systemd-binfmt.service,ipset.service,remote-fs-pre.target,umount.target,boot.mount,veritysetup-pre.target,dracut-mount.service,dracut-cmdline.service,sshd-keygen.service,initrd-switch-root.service,fstrim.service,wtmpdbd.service,cryptsetup-pre.target,logrotate.service,initrd-cleanup.service,systemd-tmpfiles-clean.service,firewalld.service,systemd-networkd-wait-online.service,xdm.service"
```

