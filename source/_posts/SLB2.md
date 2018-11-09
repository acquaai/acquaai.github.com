---
title: Install HAProxy + Keepalived
date: 2017-11-11 22:03:14
categories: DevOps
---
## Preconfiguration setup

+ 关闭防火墙

```bash
$ sudo cat /etc/redhat-release 
  CentOS Linux release 7.3.1611 (Core)

$ sudo systemctl stop firewalld
$ sudo systemctl disable firewalld
$ sudo sed -i 's#^SELINUX=.*#SELINUX=disabled#' /etc/sysconfig/selinux
$ sudo setenfore 0
```

+ 修改主机名

```bash
$ sudo hostnamectl
$ sudo yum install -y ntp
```
+ 配置NTP服务

```bash
$ sudo systemctl start ntpd
```
<!-- more -->

## Install HAProxy

+ 创建用户、组

```bash
$ sudo useradd haproxy -s /usr/sbin/nologin
$ sudo groupadd haproxy
```

+ 安装系统包

```bash
$ sudo yum install -y gcc net-snmp net-snmp-devel net-snmp-agent-libs net-snmp-libs  pcre-static pcre-devel
```

+ Download [HAProxy](http://www.haproxy.org/)

```bash
$ tar xzvf haproxy-x.x.x.tar.gz
$ cd haproxy-x.x.x
```

+ 编译安装

```bash
$ more README
$ sudo make TARGET=linux2628 PREFIX=/usr/local/haproxy
$ sudo make install PREFIX=/usr/local/haproxy
```

+ errorfile错误文件

```bash
$ sudo cp -r examples/errorfiles/  /usr/local/haproxy/
```

+ haproxy日志文件

```bash
$ sudo mkdir -p /usr/local/haproxy/log
$ sudo touch /usr/local/haproxy/log/haproxy.log
$ sudo ln -s /usr/local/haproxy/log/haproxy.log  /var/log/
```
+ etc中的haproxy文件

```bash
$ sudo mkdir -p /etc/haproxy
$ sudo ln -s /usr/local/haproxy/haproxy.cfg /etc/haproxy/
```

+ 设置全局启动文件

```bash
$ sudo ln -s /usr/local/haproxy/sbin/haproxy  /usr/sbin/haproxy
```

+ 配置HAProxy服务

```bash
$ sudo cp examples/haproxy.init /etc/rc.d/init.d/haproxy
$ sudo chmod 755 /etc/rc.d/init.d/haproxy
$ sudo systemctl daemon-reload
$ sudo chkconfig --add haproxy
$ sudo chkconfig haproxy on
$ sudo systemctl start haproxy
$ sudo systemctl status haproxy
```

## Install Keepalived

+ 创建用户、组

```bash
$ sudo useradd keepalived -s /usr/sbin/nologin
$ sudo groupadd keepalived
```

+ 开启系统网络转发功能

```bash
$ sudo echo "net.ipv4.ip_nonlocal_bind = 1" >> /etc/sysctl.conf
$ sudo echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
$ sudo sysctl -p
```

+ 安装系统包

```bash
$ sudo yum install -y openssl-devel libnl3-devel ipset-devel iptables-devel libnfnetlink-devel
```

+ Download [Keepalived](http://keepalived.org/)

```bash
$ tar xzvf keepalived-x.x.x.tar.gz
$ cd keepalived-x.x.x
$ more INSTALL
```

+ 编译安装

```bash
$ sudo ./configure
$ sudo make
$ sudo make install
```

+ 配置Keepalived服务

```bash
$ sudo cp keepalived/etc/init.d/keepalived.rh.init  /etc/init.d/keepalived
$ sudo chmod +x /etc/init.d/keepalived
$ sudo systemctl daemon-reload
$ sudo chkconfig --add keepalived
$ sudo chkconfig keepalived on
$ sudo systemctl start keepalived
$ sudo systemctl status keepalived
```
