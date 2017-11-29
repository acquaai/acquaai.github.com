---
title: Consistent Network Device Naming
date: 2017-11-29 16:38:19
categories: DevOps
---
[Naming Schemes Hierarchy](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/networking_guide/ch-consistent_network_device_naming#sec-Naming_Schemes_Hierarchy)

* Add "net.ifnames=0" and "biosdevname=0" as kernel arguments to grub
* In '/etc/sysconfig/network-scripts/' Change your configured NIC config file to 'ifcfg-ethX'
* If you have multiple interfaces and want to control naming of each device rather than letting the kernel do in its own way, /etc/udev/rules.d/60-net.rules seems necessary to override /usr/lib/udev/rules.d/60-net.rules.

``` bash
$ vi /etc/default/grub
...
GRUB_CMDLINE_LINUX="XXX quiet net.ifnames=0 biosdevname=0"
...

$ grub2-mkconfig -o /boot/grub2/grub.cfg   
```
<!-- more -->

``` bash
$ cd  /etc/sysconfig/network-scripts/
$ mv ifcfg-eno1 ifcfg-eth0

$ vi ifcfg-eth0
NAME=eth0 
DEVICE=eth0
```

``` bash
$ cat /etc/udev/rules.d/60-net.rules
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="94:57:a5:53:f2:78", ATTR{type}=="1", KERNEL=="eth*", NAME="eth0"
...
...

$ shutdown -r now
```
