---
title: ntpd vs ntpdate
date: 2017-10-28 18:40:48
categories: DevOps
comments: false
---
>10月16日[PingCAP](https://pingcap.com)正式发布了TiDB 1.0版本，作为数据库的爱好者第一时间感受一下。在部署8节点的TiDB时，TiDB会precheck所有节点的时钟同步，而precheck的方式是**only ntpd**，由于OS来源于克隆模板，模板中采用了crontab的ntpdate & hwclock -w方式来同步时间，so the question is coming, which one and why?

The ntpd program is an operating system daemon which sets and maintains the system time of day in synchronism with Internet standard time servers. NTP provides a `smooth` time correction.

ntpdate does not keep any state to perform this service for you so will not provide the same kind of accuracy. ntpdate corrects the system time `instantaneously`, which can cause problems with some software (**Especially similar to database transactions, the time can't jump back**). ntpdate can be run manually as necessary to set the host clock, or it can be run from the host startup script to set the clock at boot time. It is also possible to run ntpdate from a cron script. 

NTP daemon uses sophisticated algorithms to maximize accuracy and reliability while minimizing resource use. Finally, since ntpdate does not discipline the host clock frequency as does ntpd, the accuracy using ntpdate is limited.

<!-- more -->

>ntpd是系统同步时间的守护进程，依据偏移量来线性的调整时间，拥有更高的精度和安全性；而ntpdate简单、粗暴跳变式的立即更新时间，这在事务型的数据库应用中是致命的，上一秒你还是亿万富翁，一下秒也许你就变成了穷光蛋。

## 配置NTP Server
### 安装NTP包
``` bash
$ sudo yum update
$ sudo yum install -y ntp
```
### 修改系统时区
``` bash
$ sudo timedatectl list-timezones |grep Asia/Shanghai
Asia/Shanghai
$ sudo timedatectl set-timezone Asia/Shanghai
$ sudo timedatectl 
      Local time: Sat 2017-10-28 11:39:25 CST
  Universal time: Sat 2017-10-28 03:39:25 UTC
        RTC time: Sat 2017-10-28 03:39:25
       Time zone: Asia/Shanghai (CST, +0800)
     NTP enabled: no
NTP synchronized: no
 RTC in local TZ: no
     DST active: n/a
```
``` bash
CentOS 7之前版本（可能需要tzdata包）：
$ sudo vi /etc/sysconfig/clock 
ZONE="Asia/Shanghai"
$ sudo cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```
``` bash
修改/etc/ntp.conf
国内阿里有原子时钟，可以使用阿里公布的时钟源
$ sudo vi /etc/ntp.conf
driftfile /var/lib/ntp/drift  #以driftfile记录时间差异，百万分之一秒（ppm）
restrict default kod nomodify notrap nopeer noquery	#拒绝IPv4的连接
restrict -6 default kod nomodify notrap nopeer noquery	##拒绝IPv6的连接

nomodify: 客户端不能使用ntpc与ntpq来修改服务器的时间参数，但客户端仍可以校时
notrap: 不提供trap远程事件登录（remote event logging）的功能
noquery: 客户端不能够使用ntpq、ntpc等指令来查询时间服务器，客户端不可以校时
ignore: 拒绝所有类型的NTP连接
notrust: 拒绝没有认证的客户端

restrict 127.0.0.1	#允许本机来源
restrict -6 ::1
restrict 192.168.1.0 mask 255.255.255.0 nomodify   #允许的网段

server  time1.aliyun.com
server  time2.aliyun.com
server  time3.aliyun.com
server  time4.aliyun.com
server  time5.aliyun.com
server  time6.aliyun.com  prefer
server  time7.aliyun.com

includefile /etc/ntp/crypto/pw
keys /etc/ntp/keys
```
### 启动NTP并检查
``` bash
$ sudo systemctl start ntpd
$ ntpstat
**synchronised** to NTP server (115.28.122.198) at stratum 3 
   time correct to within 75 ms
   polling server every 1024 s

$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*time6.aliyun.co 10.137.38.86     2 u  383 1024  377   24.056   -0.829   0.594
+time4.aliyun.co 10.137.38.86     2 u  721 1024  377   20.286   -0.590   0.809
-time5.aliyun.co 10.137.38.86     2 u  229 1024  373   34.518   -4.825   2.597
+120.25.115.19   10.137.38.86     2 u  678 1024  337   18.701   -0.811   1.965

$ sudo systemctl enable ntpd
```

## 配置NTP Client
### 安装NTP包
### 修改系统时区
### 修改/etc/ntp.conf
``` bash
$ sudo vi /etc/ntp.conf
restrict 127.0.0.1
server 192.168.1.118

$ sudo systemctl start ntpd
$ sudo ntpdate 192.168.1.118	#由于client/server之间的误差不允许超过 1000 秒，可能需要先手动同步
$ ntpstat 
synchronised to NTP server (192.168.1.118) at stratum 4 
   time correct to within 79 ms
   polling server every 64 s
$ sudo systemctl enable ntpd
```
