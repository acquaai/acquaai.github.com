---
title: Install LVS + Keepalived
date: 2017-11-11 23:34:23
categories: DevOps
---
## Preconfiguration setup
**略**
## Install LVS
<!-- more -->
+ 安装系统包

```bash
$ sduo gcc kernel-devel openssl-devel libnl3-devel ipset-devel iptables-devel libnfnetlink-devel net-snmp-devel

$ sudo yum install -y ipvsadm
```

+ 安装、配置Keepalived
**略**

## LVS Configure

```bash
$ sudo vi /etc/keepalived/keepalived.conf
global_defs {
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    interface ens160
    state MASTER            // state BACKUP
    virtual_router_id 51
    priority 100            // priority 99 on BACKUP
    unicast_src_ip x.x.x.76
    unicast_peer {
        x.x.x.77
    }
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        x.x.x.78
    }
}

virtual_server x.x.x.78 53 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    nat_mask 255.255.255.0
    persistence_timeout 50
    protocol UDP

    real_server x.x.x.73 53 {
        weight 100
            TCP_CHECK {
                    connect_timeout 3
                    nb_get_retry 3
                    delay_before_retry 3
                    connect_port 53
        }
    }
    real_server x.x.x.74 53 {
        weight 100
            TCP_CHECK {
                    connect_timeout 3
                    nb_get_retry 3
                    delay_before_retry 3
                    connect_port 53
        }
    }
}
```

## RealServer Configure

+ Linux Platform

```bash
在 lo 适配器配置vip

$ sudo vi /etc/sysconfig/network-scripts/ifcfg-lo:0
DEVICE=lo:0
IPADDR=x.x.x.78
NETMASK=255.255.255.255
ONBOOT=yes
NAME=loopback

$ sudo ifup lo:0

禁止广播

$ sudo vi /etc/sysctl.conf
net.ipv4.conf.lo.arp_ignore = 1      // 只回答目标IP地址是来访网络接口本地地址的ARP查询请求
net.ipv4.conf.lo.arp_announce = 2    // 忽略IP数据包的源地址并尝试选择与能该地址通信的本地址
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2

$ sudo sysctl -p
```

> + arp_ignore:定义对目标地址为本地IP的ARP询问不同的应答模式0 0 - (默认值): 回应任何网络接口上对任何本地IP地址的arp查询请求 1 - 只回答目标IP地址是来访网络接口本地地址的ARP查询请求 2 -只回答目标IP地址是来访网络接口本地地址的ARP查询请求,且来访IP必须在该网络接口的子网段内 3 - 不回应该网络界面的arp请求，而只对设置的唯一和连接地址做出回应 4-7 - 保留未使用 8 -不回应所有（本地地址）的arp查询。
> 
> + arp_announce:对网络接口上，本地IP地址的发出的，ARP回应，作出相应级别的限制: 确定不同程度的限制,宣布对来自本地源IP地址发出Arp请求的接口 0 - (默认) 在任意网络接口（eth0,eth1，lo）上的任何本地地址 1 -尽量避免不在该网络接口子网段的本地地址做出arp回应. 当发起ARP请求的源IP地址是被设置应该经由路由达到此网络接口的时候很有用.此时会检查来访IP是否为所有接口上的子网段内ip之一.如果改来访IP不属于各个网络接口上的子网段内,那么将采用级别2的方式来进行处理 2 - 对查询目标使用最适当的本地地址.在此模式下将忽略这个IP数据包的源地址并尝试选择与能与该地址通信的本地地址.首要是选择所有的网络接口的子网中外出访问子网中包含该目标IP地址的本地地址. 如果没有合适的地址被发现,将选择当前的发送网络接口或其他的有可能接受到该ARP回应的网络接口来进行发送。

```bash
自定义脚本可达到相同效果

#!/usr/bin/bash
. /etc/rc.d/init.d/functions
 VIP=x.x.x.78

case "$1" in
start)
    echo "start LVS of Realserver DR mode"
    /usr/sbin/ifconfig lo:0  $VIP broadcast $VIP netmask 255.255.255.255 up
    echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
    echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
    echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
    echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
    ;;
stop)
    /usr/sbin/ifconfig lo:0 down
    echo "stop LVS of Realserver DR mode"
    echo "0" >/proc/sys/net/ipv4/conf/lo/arp_ignore
    echo "0" >/proc/sys/net/ipv4/conf/lo/arp_announce
    echo "0" >/proc/sys/net/ipv4/conf/all/arp_ignore
    echo "0" >/proc/sys/net/ipv4/conf/all/arp_announce
    ;;
   *)
   echo "Usage: $0 {start|stop}"
   exit 1
esac
```

+ Windows Platform

> + 配置回环网络适配器 打开设备管理器，点中网络适配器，然后点击操作，选择里面的“添加过时硬件”； 弹出的窗口单击下一步，然后选择“安装我手动从列表选择的硬件”，下一步； 弹出窗口右边滚动条拉到底，选择网络适配器，下一步； 厂商选择Microsoft，网络适配器选择Loopback Adapter (其它系统可能是KM-TEST），下一步；等到系统安装环回网卡，安装好了点击完成。

> + 配置回环网络适配器IP IP: x.x.x.78 Netmask: 255.255.255.255

> + 设置回环网络适配器的工作模式 Windows系统中，网卡有两种工作模式：强工作模式（stronghost）和弱工作模式（weakhost）。在强工作模式下，正常网卡和回环网卡之间是不能进行数据转发的，数据转发只能工作在弱工作模式下，如果没有设置弱工作模式的话，使用VIP地址不能访问DNS/WEB服务器，使用ipvsadm -lcn查看连接状态时，会发现所有的连接状态都是SYN_RECV状态。 以管理员身份打开命令提示符，设置弱工作模式命令如下：

```bash
CMD: 
netsh interface ipv4 set interface "Ethernet0" weakhostreceive=enabled
netsh interface ipv4 set interface "Ethernet0" weakhostsend=enabled
netsh interface ipv4 set interface "loop0" weakhostreceive=enabled
netsh interface ipv4 set interface "loop0" weakhostsend=enabled
```

> 关于 LVS 工作模式的切换可[参考](https://blog.gnuers.org/?p=647)

## LVS管理工具 - ipvsman

### Install ipvsman

+ 安装系统包

```bash
$ sudo yum -y install python-dns python-urwid
```

+ 安装[python-egenix-mxdatetime](https://www.egenix.com/products/python/mxBase/mxDateTime/)包

```bash
$ unzip egenix-mx-base-3.2.9-py2.7_ucs4-linux-x86_64-prebuilt.zip
$ cd egenix-mx-base-3.2.9-py2.7_ucs4-linux-x86_64-prebuilt
$ sudo python setup.py install
```

+ Download [ipvsman](https://sourceforge.net/projects/ipvsman/files/ipvsman/)

```bash
$ tar xzvf ipvsman_0.9.2-10.tar.gz
$ cd trunk
$ sudo python setup.py install
```

### ipvsman Configure

```bash
$ vi /etc/ipvsman/topology.cfg
virtual_server {
       label DNSService
       ip x.x.x.78
       port 53
       delay_loop 6
       lb_algo rr
       lb_kind DR
       persistence_timeout 50
       protocol UDP

       sorry_server {
          label SORRY_Server
          ip x.x.x.108
          port 53
          }
          
       real_server {
           label DNS1
           ip x.x.x.73
           port 53
           weight 100
           TCP_CHECK {
               connect_timeout 3
           }
       }
       real_server {
           label DNS2
           ip x.x.x.74
           port 53
           weight 100
           TCP_CHECK {
               connect_timeout 3
           }
       }
   }
```

### Start ipvsman

```bash
$ sudo ipvsman
```
