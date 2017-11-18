---
title: ipvs-fullnat-and-synproxy
tags:
---
## Building Kernel
### SYSINFO

``` bash
$ cat /etc/redhat-release 
CentOS release 6.4 (Final)
$ uname -a
Linux FULLNAT 2.6.32-358.el6.x86_64 #1 SMP Fri Feb 22 00:31:26 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux
```

### Install RPMs-based

``` bash
$ yum -y install m4 gcc bison redhat-rpm-config xmlto asciidoc elfutils-libelf-devel binutils-devel newt-devel perl-ExtUtils-Embed hmaccalc rng-tools patchutils python-devel rpm-build wget
```

### Get kernel source

```bash
$ mkdir /home/acqua
$ cd acqua
$ wget http://vault.centos.org/6.4/os/Source/SPackages/kernel-2.6.32-358.el6.src.rpm

$  cat > ~/.rpmmacros << EOF
%_topdir /home/acqua/rpms
%_tmppath /home/acqua/rpms/tmp
%_sourcedir /home/acqua/rpms/SOURCES
%_specdir /home/acqua/rpms/SPECS
%_srcrpmdir /home/acqua/rpms/SRPMS
%_rpmdir /home/acqua/rpms/RPMS
%_builddir /home/acqua/rpms/BUILD
EOF
<!-- more -->

$ cd /home/acqua
$ mkdir rpms
$ mkdir rpms/tmp
$ mkdir rpms/SOURCES
$ mkdir rpms/SPECS
$ mkdir rpms/SRPMS
$ mkdir rpms/RPMS
$ mkdir rpms/BUILD
$ groupadd  mockbuild
$ useradd -g mockbuild -s /sbin/nologin -M mockbuild
$ rpm -ivh kernel-2.6.32-358.el6.src.rpm

$ cd /home/acqua/rpms/SOURCES
$ vi config-generic

`line1 # x86_64`
`line797 CONFIG_IP_VS_TAB_BITS=20`

$ cd /home/acqua/rpms/SPECS/
$ vi kernel.spec
`line17 %define buildid .ipvs_20bit`

$ rpmbuild -bp kernel.spec

>  If this takes a long time, you might wish to run rngd in the background to keep the supply of entropy topped up.  It needs to be run as root, and should use a hardware random number generator if one is available, eg:
> `rngd -r /dev/hwrandom`

> If one isn't available, the pseudo-random number generator can be used:
> `rngd -r /dev/urandom`

kernel source location
$ ll /home/acqua/rpms/BUILD
drwxr-xr-x. 4 root root 4096 Nov 19 07:22 kernel-2.6.32-358.el6
```

## Apply Patch

``` bash

$ cd /home/acqua
$ wget http://kb.linuxvirtualserver.org/images/a/a5/Lvs-fullnat-synproxy.tar.gz
$ tar xzvf Lvs-fullnat-synproxy.tar.gz

$ cd /home/acqua/rpms/BUILD/kernel-2.6.32-358.el6/linux-2.6.32-358.el6.ipvs_20bit.x86_64
$ cp /home/acqua/lvs-fullnat-synproxy/lvs-2.6.32-220.23.1.el6.patch ./
$ cp /home/acqua/lvs-fullnat-synproxy/toa-2.6.32-220.23.1.el6.patch ./

$ cp /home/acqua/ip_vs.h /home/acqua/rpms/BUILD/kernel-2.6.32-358.el6/linux-2.6.32-358.el6.ipvs_20bit.x86_64/include/net/

$ cd /home/acqua/rpms/BUILD/kernel-2.6.32-358.el6/linux-2.6.32-358.el6.ipvs_20bit.x86_64/net/netfilter/ipvs/
$ rm -rf ./*

$ tar xzvf ipvs.tar.gz 
$ cp /home/acqua/ipvs/* /home/acqua/rpms/BUILD/kernel-2.6.32-358.el6/linux-2.6.32-358.el6.ipvs_20bit.x86_64/net/netfilter/ipvs/

$ cd /home/acqua/rpms/BUILD/kernel-2.6.32-358.el6/linux-2.6.32-358.el6.ipvs_20bit.x86_64/ 
$ patch -p1 < ./lvs-2.6.32-220.23.1.el6.patch
$ patch -p1 < ./toa-2.6.32-220.23.1.el6.patch
```

```bash
$ cp configs/kernel-2.6.32-x86_64.config .config
$ vi Makefile
`line4 EXTRAVERSION = -358.e16.lvs-fullnat`

$ make -j4
$ make modules_install
$ make install

$ vi /etc/grub.conf

不关闭nohz，大压力下CPU0可能会消耗过高，压力不均匀
```

## LVS Tools
### 加载驱动

``` bash
$ modprobe ip_vs
$ modprobe ip_vs_rr
$ modprobe ip_vs_wrr
$ modprobe ip_vs_sh
$ modprobe iptable_filter
$ modprobe ip_tables
$ modprobe toa
```

### Install Tools

``` bash
$ cd /home/acqua/lvs-fullnat-synproxy
$ tar xzf lvs-tools.tar.gz
$ cd tools/keepalived/
$ ./configure --with-kernel-dir=”/lib/modules/`uname -r`/build”
$ make
$ make install

$ cd ../ipvsadm/
$ make
$ make install

quagga可以 yum -y install quagga安装也可以编译安装
$ cd ../quagga
$ ./configure --disable-ripd --disable-ripngd --disable-bgpd --disable-watchquagga --disable-doc  --enable-user=root --enable-vty-group=root --enable-group=root --enable-zebra --localstatedir=/var/run/quagga
$ make
$ make install
```

## Tunning

``` bash
$ cat > tunning.sh << EOF
ethtool -K p4p1 gro off
ethtool -K p4p1 lro off
ethtool -K p4p1 rx off && sleep 1 && ethtool -K p4p1 rx on &
echo 1 > /proc/sys/net/ipv4/tcp_syncookies
echo 500000 > /proc/sys/net/core/netdev_max_backlog
echo 500000 > /proc/sys/net/ipv4/tcp_max_syn_backlog
echo 1 > /proc/sys/net/ipv4/ip_forward
echo 0 > /proc/sys/net/ipv4/conf/all/send_redirects
echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
echo 2000 > /proc/sys/net/unix/max_dgram_qlen
EOF

$ cat tunning.sh >> /etc/rc.local
```

**Reference**
[LVS Official Website](http://kb.linuxvirtualserver.org/wiki/IPVS_FULLNAT_and_SYNPROXY)
[云栖社区: 从一个开发的角度看负载均衡和LVS](https://yq.aliyun.com/articles/26557)
[阿里维护: SLB技术原理浅析](http://www.aliweihu.com/24.html)









https://jeffrycheng.com/2017/03/31/在R730上安装lvs-fullnat/

drivers/md/dm-bufio.ko.unsigned:(.gnu.linkonce.this_module+0x0): multiple definition of `no symbol'
drivers/md/dm-bufio.ko.unsigned: In function `no symbol':
/home/acqua/rpms/BUILD/kernel-2.6.32-358.el6/linux-2.6.32-358.el6.ipvs_20bit.x86_64/drivers/md/dm-bufio.c:1263: multiple definition of `no symbol'
ld: ld: BFD version 2.20.51.0.2-5.36.el6 20100205 internal error, aborting at bfd.c line 628 in _bfd_default_error_handler

ld: Please report this bug.

make[1]: *** [drivers/md/dm-bufio.ko] Error 1
make[1]: *** Waiting for unfinished jobs....
drivers/md/dm-delay.ko.unsigned: In function `no symbol':
(.exit.text+0x0): multiple definition of `no symbol'
drivers/md/dm-delay.ko.unsigned:(.gnu.linkonce.this_module+0x0): first defined here
ld: Warning: size of symbol `' changed from 560 in drivers/md/dm-delay.ko.unsigned to 47 in drivers/md/dm-delay.ko.unsigned
ld: Warning: type of symbol `' changed from 1 to 2 in drivers/md/dm-delay.ko.unsigned
drivers/md/dm-delay.ko.unsigned: In function `no symbol':
(.init.text+0x0): multiple definition of `no symbol'
drivers/md/dm-delay.ko.unsigned:(.gnu.linkonce.this_module+0x0): first defined here
ld: Warning: size of symbol `' changed from 47 in drivers/md/dm-delay.ko.unsigned to 190 in drivers/md/dm-delay.ko.unsigned
make[1]: *** [drivers/md/dm-delay.ko] Error 1
make[1]: *** wait: No child processes.  Stop.
make: *** [modules] Error 2

