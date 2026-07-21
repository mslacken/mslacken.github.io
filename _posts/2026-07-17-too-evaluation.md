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
The standard GO implementation of MPC at https://github.com/modelcontextprotocol/go-sdk doesn't offer toon yet, I used https://github.com/aj-geddes/toon-context-mcp . The implementation is relatively straight forward as the real payload of a MCP is text field which contains the as json marshalled internal structure. So this text field must now marshalled using toon. The outer json structure which is used for communication between the MCP server and the client is left intact.

## Test bed

Creating the test bed is a bit more complicated and needs some refining. At first the MCP server needs a real, life running systemd process to connect to. So a reproducible and secure solution for this is to use virtual machine. The tofu (OSS terraform) definition which can be found at https://github.com/mslacken/EvalToon/blob/main/tofu/leap-cloud.tf which uses an openSUSE Tumbleweed cloud image as it can be configured via cloud-init.
A note able side quest here was to get the systemd-mcp server into the VM, without compiling it the whole would consume some CPU cycles and using the rpm package won't work as I didn't push the toon changes upstream, yet. The canonical way to copy now a binary to a VM would be use simply copy the file VM with `scp`. I didn't want to use this method as the ssh server was disabled and fixing this, I wanted to have as task to the LLM. This leaves following possibilities (or some more):
* use 9p or similar FS
* use a readonly FS containing only the binary
As the 9p solution would require abolute paths in the cloud init config, I just created a second virtual disk which just contains the binary. After the start of the VM I could just `dd` to dump the binary into the FS.
The setup script additionally stopped the `ssh` server so that this "fail" could be used as test bed.

## Agent 

Now we need the connection between the MCP server and the LLM which was running on a remote ollama instance. For this I chose the google agent development kit https://adk.dev which wrapped all this tasks.
Althouhg ADK has the possibly for external logging, it can also log the events to a local sqlite database what is important as we can read out from this database the relevant measurements the variables which where, the number of tokens presented to the LLM, the number of tokens created by the LLM, number of tool calls and total number of ollama calls.

## LLM

The external ollama instance ran a `gemma4` model with `31.3B` parameters and `Q4_K_M` quantization, a temperature of 1 and the context length was `131072` tokens.


## Experiments

Following three queries were presented to the LLM
* 'Check if sshd is running' labeled as **CHECK**
* 'List me running services on the system and which services can be started' labeled as **LIST**
* 'I can't login, fix this!' labeled as **FIX**

The first two quest should be easy to solve for the LLM and just result in some token calls, as the last one should require a lot more tool calls and so test if toon works in real world scenarios with complex tool calls.
For every query I made 30 runs with and without toon

## Results

The results can be summarized in following table

| Query | Mean tokens | Median tokens | Mean created tokens| Mean tool calls
| ------------- | -------------- | -------------- | -------------- |

| **CHECK** | 10621 | 11793 | 258 |1.70 |
|**CHECK_TOON**| 10337 | 11755| 239 | 1.63 |
| ------------- | -------------- | -------------- | -------------- |

| **LIST**|10827 | 12333 | 1747 |2.07 |
| **LIST_TOON** | 11846 | 12364 |1751 | 2.10 |
| ------------- | -------------- | -------------- | -------------- |

| **FIX** | 36738 | 26735 | 871 |6.63 |
| **FIX_TOON** | 32466 | 26816 |766 | 5.97 |

As you can see the using toon reduces as well the tokens presented to the LLM as well as the number of tokens created by the LLM. It's a bit suprising that if toon is used always less tokens are created, but it seems to be a trend that the number of created tokens depend on the number of presented tokens.
For the simple example where the system was just tasked to check if ssh is running using toon doesn't make any big difference, asd most of the tokens are used up by the system prompt and the description of the tools available.
For the listing of the services, what is basically a huge list, using toon performed worse as for that query slightly more tool calls were done if toon was used. Hence the number of created tokens was increased.
![Number of tool and ollama calls of the **FIX** task with some outliers][{{site.url}}/assets/tokens_login_fix_calls.png]
![Number of tool and ollama calls of the **FIX_TOON** task without outliers][{{site.url}}/assets/tokens_toon_login_fix_calls.png]
![Number of the genberated tokens for **FIX** where also the outliers are visible][{{site.url}}/assets/tokens_login_fix_cand_tokens.png]
![Number of the genberated tokens for **FIX_TOON** with a more compact distribution][{{site.url}}/assets/tokens_login_fix_cand_tokens.png]
For the most complex task which requires the LLM identify the stopped ssh service and start it, the more compact results from the tools make the biggest effect, not just in tne numbers presented to the LLM, but also in a more efficient tool usage due to higher number of tasks which got of the rails if no toon was used.
Not using pure json doesn't hinder the LLM, but the efficient use of the context window seems here superior.

## Summary

Although using toon for small task doesn't seem to make a big difference as most of the tokens are spent on the system prompt and MCP server description, it performs superior if used for complex tasks as it seems to keep the LLMs more focused. Most likely this is due to the fact, that information density is higher which leads to a higher performance for complex tasks.


## Links

* https://github.com/mslacken/EvalToon
* https://github.com/openSUSE/systemd-mcp


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


## Raw data

### CHECK

```
# query prom_tokens cand_tokens tot_tokens tool_calls ollama_calls
"Check if sshd is running" 11936 289 12225 2 3
"Check if sshd is running" 7732 143 7875 1 2
"Check if sshd is running" 11788 257 12045 2 3
"Check if sshd is running" 11785 250 12035 2 3
"Check if sshd is running" 12056 379 12435 2 3
"Check if sshd is running" 11875 310 12185 2 3
"Check if sshd is running" 11802 238 12040 2 3
"Check if sshd is running" 11819 277 12096 2 3
"Check if sshd is running" 11825 285 12110 2 3
"Check if sshd is running" 11807 266 12073 2 3
"Check if sshd is running" 11909 325 12234 2 3
"Check if sshd is running" 11788 248 12036 2 3
"Check if sshd is running" 11786 275 12061 2 3
"Check if sshd is running" 11754 211 11965 2 3
"Check if sshd is running" 7786 195 7981 1 2
"Check if sshd is running" 7736 215 7951 1 2
"Check if sshd is running" 12026 597 12623 2 3
"Check if sshd is running" 11778 263 12041 2 3
"Check if sshd is running" 7827 225 8052 1 2
"Check if sshd is running" 7733 138 7871 1 2
"Check if sshd is running" 11843 284 12127 2 3
"Check if sshd is running" 11798 248 12046 2 3
"Check if sshd is running" 7719 132 7851 1 2
"Check if sshd is running" 7856 255 8111 1 2
"Check if sshd is running" 11901 309 12210 2 3
"Check if sshd is running" 11823 262 12085 2 3
"Check if sshd is running" 11836 284 12120 2 3
"Check if sshd is running" 7733 151 7884 1 2
"Check if sshd is running" 11861 301 12162 2 3
"Check if sshd is running" 7735 143 7878 1 2
```


### CHECK_TOON

```
 query prom_tokens cand_tokens tot_tokens tool_calls ollama_calls
"Check if sshd is running" 11843 283 12126 2 3
"Check if sshd is running" 7831 241 8072 1 2
"Check if sshd is running" 11842 305 12147 2 3
"Check if sshd is running" 7801 213 8014 1 2
"Check if sshd is running" 11753 237 11990 2 3
"Check if sshd is running" 11742 216 11958 2 3
"Check if sshd is running" 11825 282 12107 2 3
"Check if sshd is running" 7809 221 8030 1 2
"Check if sshd is running" 7774 187 7961 1 2
"Check if sshd is running" 11824 264 12088 2 3
"Check if sshd is running" 7736 148 7884 1 2
"Check if sshd is running" 11789 248 12037 2 3
"Check if sshd is running" 11980 351 12331 2 3
"Check if sshd is running" 11778 247 12025 2 3
"Check if sshd is running" 11766 208 11974 2 3
"Check if sshd is running" 7842 248 8090 1 2
"Check if sshd is running" 11770 234 12004 2 3
"Check if sshd is running" 7755 163 7918 1 2
"Check if sshd is running" 11730 210 11940 2 3
"Check if sshd is running" 11809 257 12066 2 3
"Check if sshd is running" 11835 279 12114 2 3
"Check if sshd is running" 7843 240 8083 1 2
"Check if sshd is running" 11850 288 12138 2 3
"Check if sshd is running" 11820 264 12084 2 3
"Check if sshd is running" 7786 199 7985 1 2
"Check if sshd is running" 7757 165 7922 1 2
"Check if sshd is running" 7820 217 8037 1 2
"Check if sshd is running" 11758 236 11994 2 3
"Check if sshd is running" 11897 306 12203 2 3
"Check if sshd is running" 11750 226 11976 2 3
```


### **LIST**

```
# query prom_tokens cand_tokens tot_tokens tool_calls ollama_calls
"List me running services on the system and which services can be started" 12347 725 13072 2 2
"List me running services on the system and which services can be started" 12437 864 13301 2 2
"List me running services on the system and which services can be started" 12421 2120 14541 2 2
"List me running services on the system and which services can be started" 12429 2683 15112 2 2
"List me running services on the system and which services can be started" 12398 2366 14764 2 2
"List me running services on the system and which services can be started" 8976 2675 11651 2 2
"List me running services on the system and which services can be started" 12400 2119 14519 2 2
"List me running services on the system and which services can be started" 12449 1946 14395 2 2
"List me running services on the system and which services can be started" 8947 2458 11405 2 2
"List me running services on the system and which services can be started" 8902 2481 11383 2 2
"List me running services on the system and which services can be started" 8884 1303 10187 2 2
"List me running services on the system and which services can be started" 8898 1386 10284 2 2
"List me running services on the system and which services can be started" 8953 2538 11491 2 2
"List me running services on the system and which services can be started" 12418 829 13247 2 2
"List me running services on the system and which services can be started" 9046 1268 10314 2 2
"List me running services on the system and which services can be started" 9343 984 10327 3 2
"List me running services on the system and which services can be started" 12532 2220 14752 2 2
"List me running services on the system and which services can be started" 8934 1609 10543 2 2
"List me running services on the system and which services can be started" 12364 869 13233 2 2
"List me running services on the system and which services can be started" 8990 1816 10806 2 2
"List me running services on the system and which services can be started" 12399 2011 14410 2 2
"List me running services on the system and which services can be started" 8883 924 9807 2 2
"List me running services on the system and which services can be started" 8968 2432 11400 2 2
"List me running services on the system and which services can be started" 12343 2084 14427 2 2
"List me running services on the system and which services can be started" 12405 1481 13886 2 2
"List me running services on the system and which services can be started" 12342 1761 14103 2 2
"List me running services on the system and which services can be started" 12450 727 13177 2 2
"List me running services on the system and which services can be started" 12324 582 12906 2 2
"List me running services on the system and which services can be started" 9472 2749 12221 3 2
"List me running services on the system and which services can be started" 9184 2423 11607 3 2
```

### LIST_TOON

```
# query prom_tokens cand_tokens tot_tokens tool_calls ollama_calls
"List me running services on the system and which services can be started" 12341 2520 14861 2 2
"List me running services on the system and which services can be started" 12362 621 12983 2 2
"List me running services on the system and which services can be started" 12384 1374 13758 2 2
"List me running services on the system and which services can be started" 12326 1977 14303 2 2
"List me running services on the system and which services can be started" 8987 2383 11370 3 2
"List me running services on the system and which services can be started" 8914 2509 11423 2 2
"List me running services on the system and which services can be started" 12389 2127 14516 2 2
"List me running services on the system and which services can be started" 12568 789 13357 2 2
"List me running services on the system and which services can be started" 12376 1963 14339 2 2
"List me running services on the system and which services can be started" 8885 1814 10699 2 2
"List me running services on the system and which services can be started" 12481 1995 14476 2 2
"List me running services on the system and which services can be started" 12366 2294 14660 2 2
"List me running services on the system and which services can be started" 8875 1692 10567 2 2
"List me running services on the system and which services can be started" 9034 2344 11378 2 2
"List me running services on the system and which services can be started" 12531 2390 14921 2 2
"List me running services on the system and which services can be started" 12335 1998 14333 2 2
"List me running services on the system and which services can be started" 12444 684 13128 2 2
"List me running services on the system and which services can be started" 12445 1775 14220 2 2
"List me running services on the system and which services can be started" 29548 857 30405 3 4
"List me running services on the system and which services can be started" 12331 1720 14051 2 2
"List me running services on the system and which services can be started" 8982 2211 11193 2 2
"List me running services on the system and which services can be started" 12477 2182 14659 2 2
"List me running services on the system and which services can be started" 12400 1448 13848 2 2
"List me running services on the system and which services can be started" 9244 1857 11101 2 2
"List me running services on the system and which services can be started" 8882 825 9707 2 2
"List me running services on the system and which services can be started" 9078 2073 11151 2 2
"List me running services on the system and which services can be started" 12549 936 13485 2 2
"List me running services on the system and which services can be started" 12466 1201 13667 2 2
"List me running services on the system and which services can be started" 9021 1732 10753 2 2
"List me running services on the system and which services can be started" 12377 2267 14644 2 2
```


### FIX

```
"I can't login, fix this!" 41367 964 42331 8 9
"I can't login, fix this!" 25606 619 26225 5 6
"I can't login, fix this!" 22251 816 23067 6 5
"I can't login, fix this!" 22130 613 22743 4 5
"I can't login, fix this!" 26780 748 27528 5 6
"I can't login, fix this!" 112901 943 113844 9 10
"I can't login, fix this!" 54626 753 55379 7 8
"I can't login, fix this!" 25405 547 25952 5 6
"I can't login, fix this!" 16866 615 17481 4 4
"I can't login, fix this!" 59437 958 60395 7 8
"I can't login, fix this!" 33176 929 34105 6 7
"I can't login, fix this!" 16453 489 16942 3 4
"I can't login, fix this!" 26306 611 26917 5 6
"I can't login, fix this!" 22935 913 23848 5 5
"I can't login, fix this!" 84688 2655 87343 13 8
"I can't login, fix this!" 25757 641 26398 5 6
"I can't login, fix this!" 31397 853 32250 6 7
"I can't login, fix this!" 26691 695 27386 5 6
"I can't login, fix this!" 27273 794 28067 5 6
"I can't login, fix this!" 94618 2614 97232 15 16
"I can't login, fix this!" 31886 777 32663 6 7
"I can't login, fix this!" 25962 551 26513 5 6
"I can't login, fix this!" 26899 745 27644 5 6
"I can't login, fix this!" 25976 662 26638 5 6
"I can't login, fix this!" 25531 537 26068 5 6
"I can't login, fix this!" 54975 793 55768 7 8
"I can't login, fix this!" 26921 748 27669 5 6
"I can't login, fix this!" 26531 778 27309 5 6
"I can't login, fix this!" 36614 712 37326 6 5
"I can't login, fix this!" 24196 1074 25270 6 5
```

### FIX_TOON

```
# query prom_tokens cand_tokens tot_tokens tool_calls ollama_calls
"I can't login, fix this!" 26317 581 26898 5 6
"I can't login, fix this!" 20708 494 21202 4 5
"I can't login, fix this!" 18571 1030 19601 7 4
"I can't login, fix this!" 30497 782 31279 6 7
"I can't login, fix this!" 22412 689 23101 5 5
"I can't login, fix this!" 22012 647 22659 6 5
"I can't login, fix this!" 37325 1379 38704 10 7
"I can't login, fix this!" 26394 651 27045 5 6
"I can't login, fix this!" 55580 956 56536 7 8
"I can't login, fix this!" 31431 680 32111 6 7
"I can't login, fix this!" 28580 820 29400 5 6
"I can't login, fix this!" 36118 839 36957 7 8
"I can't login, fix this!" 22785 770 23555 5 5
"I can't login, fix this!" 20158 346 20504 4 5
"I can't login, fix this!" 26159 567 26726 5 6
"I can't login, fix this!" 63605 875 64480 8 9
"I can't login, fix this!" 47181 935 48116 7 5
"I can't login, fix this!" 41776 856 42632 8 9
"I can't login, fix this!" 21752 585 22337 6 5
"I can't login, fix this!" 53449 707 54156 7 8
"I can't login, fix this!" 26152 730 26882 5 6
"I can't login, fix this!" 22470 704 23174 4 5
"I can't login, fix this!" 74308 1094 75402 9 10
"I can't login, fix this!" 27106 769 27875 5 6
"I can't login, fix this!" 26527 688 27215 5 6
"I can't login, fix this!" 26470 682 27152 5 6
"I can't login, fix this!" 27317 818 28135 5 6
"I can't login, fix this!" 37025 960 37985 7 8
"I can't login, fix this!" 27619 765 28384 6 6
"I can't login, fix this!" 26177 598 26775 5 6
```

