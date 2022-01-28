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
