---
title: DPVS
date: 2017-11-21 17:21:07
categories: DevOps
---
`There is no best, only better. `

[DPVS](https://github.com/iqiyi/dpvs)
![](/images/dpvs.png)

<!-- more -->

## Clone DPVS

``` bash
$ cat /etc/redhat-release 
CentOS Linux release 7.2.1511 (Core) 
$ uname -r
3.10.0-327.el7.x86_64

$ yum install -y make cmake gcc gcc-c++ patch git wget
$ git clone https://github.com/iqiyi/dpvs.git
$ cd dpvs
```

## DPDK setup

``` bash
$ wget http://fast.dpdk.org/rel/dpdk-16.07.2.tar.xz
$ tar xvf dpdk-16.07.2.tar.xz 

$ cp patch/dpdk-16.07/0001-kni-use-netlink-event-for-multicast-driver-part.patch dpdk-stable-16.07.2/

$ cd dpdk-stable-16.07.2/
$ patch -p 1 < 0001-kni-use-netlink-event-for-multicast-driver-part.patch
```

Now build DPDK and export RTE_SDK env variable for DPDK app (DPVS)

``` bash
$ make config T=x86_64-native-linuxapp-gcc
Configuration done
```

u need install **kernel-devel-`uname -r`**, or u got below error:

> make: *** /lib/modules/3.10.0-327.el7.x86_64/build: No such file or directory.  Stop.
make[5]: *** [igb_uio.ko] Error 2
make[4]: *** [igb_uio] Error 2
make[4]: *** Waiting for unfinished jobs....

``` bash

$ cd
$ wget https://buildlogs.centos.org/c7.1511.00/kernel/20151119220809/3.10.0-327.el7.x86_64/kernel-devel-3.10.0-327.el7.x86_64.rpm
$ yum install local kernel-devel-3.10.0-327.el7.x86_64.rpm
$ ln -fs /usr/src/kernels/3.10.0-327.36.1.el7.x86_64/ /lib/modules/3.10.0-327.el7.x86_64/build

logout & login

$ cd /root/dpvs/dpdk-stable-16.07.2
$ make -j 32
$ export RTE_SDK=$PWD
```

> 在本教程中，没有设置RTE_TARGET的值，默认为"build"，因为DPDK的libs与header文件在dpdk-stable-16.07.2/build中，但也可以设置为其它值。

设置DPDK hugepage, 本测试环境中使用`非统一内存访问`[NUMA](https://en.wikipedia.org/wiki/Non-uniform_memory_access)机器。For single-node system refer the [link](http://dpdk.org/doc/guides/linux_gsg/sys_reqs.html).

**Required Tools and Libraries**

 * GNU make
 * Upgrade gcc 4.9 or later

``` bash
$ wget https://mirrors.tuna.tsinghua.edu.cn/gnu/gcc/gcc-5.3.0/gcc-5.3.0.tar.gz
$ tar xzvf gcc-5.3.0.tar.gz 
$ cd gcc-5.3.0
$ ./contrib/download_prerequisites 
$ mkdir bld
$ cd bld
$ ../configure -enable-checking=release -enable-languages=c,c++ -disable-multilib
$ make -j 32
$ make install
$ gcc -v
```

* libc headers
* Linux kernel headers
* libnuma-devel-library
* Python 2.7+ or 3.2+

> $ lscpu
Architecture:          x86_64
...
NUMA node0 CPU(s):     0-7
NUMA node1 CPU(s):     8-15

``` bash
$ echo 1024 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
$ echo 1024 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages

$ mkdir /mnt/huge
$ mount -t hugetlbfs nodev /mnt/huge/
```

Install Kernel modules and bind NIC with igb_uio driver. Quick start uses only one NIC, normally we use 2 for Full-NAT cluster, even 4 for bonding mode. Assuming eth0 will be used for DPVS/DPDK, and another standalone Linux NIC for debug, for example, eth1.

``` bash
$ modprobe uio
$ cd dpvs/dpdk-stable-16.07.2/
$ insmod build/kmod/igb_uio.ko 
$ insmod build/kmod/rte_kni.ko

$ ./tools/dpdk-devbind.py --status

Network devices using DPDK-compatible driver
============================================
<none>

Network devices using kernel driver
===================================
0000:03:00.0 'VMXNET3 Ethernet Controller' if=eno16777984 drv=vmxnet3 unused=igb_uio *Active*

Other network devices
=====================
<none>

$ ifconfig eno16777984 down
$ ./tools/dpdk-devbind.py --bind=igb_uio eno16777984
```

`dpdk-devbind.py -u` can be used to unbind driver and switch it back to Linux driver like `ixgbe`. You can also use `lspci` or `ethtool -i eth0` to check the NIC PCI bus-id. Pls see [DPDK site](http://www.dpdk.org/) for details.

## Build DPVS

Set `RTE_SDK` and build it.

``` bash
$ cd /root/dpvs/dpdk-stable-16.07.2
$ export RTE_SDK=$PWD
$ cd ..
$ yum install -y openssl openssl-devel popt-devel
$ make -j 32
$ make install
```

``` bash
$ ls bin/
dpip  dpvs  ipvsadm  keepalived
```

* `dpvs` is the main program.
* `dpip` is the tool to set IP address, route, vlan, neighbor etc.
* `ipvsadm` and `keepalived` come from LVS, both are modified.

## Launch DPVS

`dpvs.conf` must be put at `/etc/dpvs.conf`, just copy it from `conf/dpvs.conf.single-nic.sample`.

``` bash
$ cp conf/dpvs.conf.single-nic.sample /etc/dpvs.conf
$ cd /root/dpvs/bin/
$ ./dpvs &
```

Check if it's get started ?

``` bash
$ ./dpip link show


```

## Test Full-NAT Load Balancer

## Configure Tutorial

[Tutorial Document](https://github.com/iqiyi/dpvs/blob/master/doc/tutorial.md)
