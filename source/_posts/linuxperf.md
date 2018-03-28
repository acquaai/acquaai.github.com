---
title: Linux性能分析
date: 2018-03-26 11:53:58
categories: DevOps
---
![](http://p564fpez5.bkt.clouddn.com/image/linuxlinux_perf_tools_full.png)

<!-- more -->

[Brendan Gregg](http://brendangregg.com) (Netflix Senior Performance Architect), 的[文章](https://medium.com/netflix-techblog/netflix-at-velocity-2015-linux-performance-tools-51964ddb81cf)讲述了 NETFLIX 大规模应用在 EC2 上的 Linux 性能分析（Use Methods）**`in 60,000 Milliseconds（1分钟）`**。

1分钟内如何做性能分析？

这就不得不提到运维人员吃饭干活，居家旅行必备的10 个命令（**`sysstat package`**）。

## 10 Commands

### uptime

了解系统负载趋势。

```bash
$ uptime
 09:36:24 up 4 days, 18:56,  1 user,  load average: 0.22, 0.15, 0.18
```

### dmesg | tail

显示系统最新的10条信息。

```bash
$ dmesg |tail
[183879.841702] device vethcb3685f entered promiscuous mode
[183879.841838] IPv6: ADDRCONF(NETDEV_UP): vethcb3685f: link is not ready
[183879.841847] docker0: port 7(vethcb3685f) entered forwarding state
[183879.841856] docker0: port 7(vethcb3685f) entered forwarding state
[183879.857773] docker0: port 7(vethcb3685f) entered disabled state
[183879.953447] IPVS: Creating netns size=2040 id=10
[183880.039549] IPv6: ADDRCONF(NETDEV_CHANGE): vethcb3685f: link becomes ready
[183880.039595] docker0: port 7(vethcb3685f) entered forwarding state
[183880.039604] docker0: port 7(vethcb3685f) entered forwarding state
[183895.093600] docker0: port 7(vethcb3685f) entered forwarding state
```

### vmstat 1

Virtual Memory Stat.

```bash
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 1393704   2104 2889104    0    0     1    21    2    6  1  0 98  0  0
 0  0      0 1393688   2104 2889140    0    0     0   119 3009 5868  1  1 98  1  0
 0  0      0 1393588   2104 2889148    0    0     0     8 2856 5327  1  1 98  0  0
 2  0      0 1393720   2104 2889148    0    0     0    68 2763 5495  1  0 99  0  0
 0  0      0 1393688   2104 2889148    0    0     0     8 2288 4503  1  0 99  0  0
 0  0      0 1393720   2104 2889148    0    0     0    56 2919 5640  1  0 99  0  0
^C
```

### mpstat -P ALL 1

查看多核CPU的负载状态，应用是否存在单线工作。

```bash
$ mpstat -P ALL 1
Linux 3.10.0-514.el7.x86_64 (n1.k8s.com)        03/26/2018      _x86_64_        (8 CPU)

09:57:37 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:57:38 AM  all    0.75    0.00    0.38    2.76    0.00    0.00    0.00    0.00    0.00   96.11
09:57:38 AM    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
09:57:38 AM    1    1.00    0.00    1.00    0.00    0.00    0.00    0.00    0.00    0.00   98.00
09:57:38 AM    2    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
09:57:38 AM    3    1.98    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   98.02
09:57:38 AM    4    1.01    0.00    0.00   22.22    0.00    0.00    0.00    0.00    0.00   76.77
09:57:38 AM    5    1.01    0.00    1.01    0.00    0.00    0.00    0.00    0.00    0.00   97.98
09:57:38 AM    6    0.00    0.00    1.01    0.00    0.00    0.00    0.00    0.00    0.00   98.99
09:57:38 AM    7    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
[...]
```

### pidstat 1

类似top，但持续输出信息，不覆盖。

```bash
$ pidstat 1
Linux 3.10.0-514.el7.x86_64 (n1.k8s.com)        03/26/2018      _x86_64_        (8 CPU)

10:02:50 AM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
10:02:51 AM     0       712    0.00    0.98    0.00    0.98     7  kube-scheduler
10:02:51 AM     0       978    0.98    0.98    0.00    1.96     7  etcd
10:02:51 AM     0      1238    0.98    0.00    0.00    0.98     7  kube-apiserver
10:02:51 AM     0      1450    0.98    0.98    0.00    1.96     6  kubelet
10:02:51 AM     0     14585    0.98    0.98    0.00    1.96     1  pidstat
10:02:51 AM     0     23261    0.98    0.00    0.00    0.98     7  fluentd

10:02:51 AM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
10:02:52 AM     0       712    1.00    0.00    0.00    1.00     2  kube-scheduler
10:02:52 AM   167       976    0.00    1.00    0.00    1.00     7  ceph-mon
10:02:52 AM     0       978    1.00    1.00    0.00    2.00     7  etcd
10:02:52 AM     0      1238    2.00    0.00    0.00    2.00     7  kube-apiserver
10:02:52 AM     0      1286    5.00    0.00    0.00    5.00     4  dockerd
10:02:52 AM     0      1301    1.00    0.00    0.00    1.00     6  docker-containe
10:02:52 AM     0      1450    5.00    0.00    0.00    5.00     6  kubelet
10:02:52 AM   999      6656    0.00    1.00    0.00    1.00     0  java
10:02:52 AM   999      6879    1.00    0.00    0.00    1.00     7  java
10:02:52 AM     0     14585    0.00    1.00    0.00    1.00     1  pidstat
[...]
```

### iostat -xz 1

`r/s，w/s，rkB/s，wkB/s:`分别表示每秒设备读次数，写次数，读的KB数，写的KB数。它们描述了磁盘的工作负载。也许性能问题就是由过高的负载所造成的。

`await:`I/O平均时间，以毫秒作单位。它是应用中I/O处理所实际消耗的时间，因为其中既包括排队用时也包括处理用时。如果它比预期的大，就意味着设备饱和了，或者设备出了问题。

`avgqu-sz:`分配给设备的平均请求数。大于1表示设备已经饱和了。（不过有些设备可以并行处理请求，比如由多个磁盘组成的虚拟设备）

`%util:`设备使用率。这个值显示了设备每秒内工作时间的百分比，一般都处于高位。低于60%通常是低性能的表现（也可以从await中看出），不过这个得看设备的类型。接近100%通常意味着饱和。

`Attention:`disk I/O性能低不一定是问题。应用的I/O往往是异步的（比如预读（read-ahead）和写缓冲（buffering for writes）），所以不一定会被阻塞并遭受延迟。

```bash
$ iostat -xz 1
Linux 3.10.0-514.el7.x86_64 (n1.k8s.com)        03/26/2018      _x86_64_        (8 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           1.04    0.00    0.43    0.14    0.00   98.38

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     2.08    0.05   13.81     4.61   163.14    24.22     0.07    5.16   15.64    5.12   1.26   1.74
dm-0              0.00     0.00    0.04   15.89     4.58   163.13    21.05     0.11    6.93   16.69    6.90   1.10   1.75
dm-1              0.00     0.00    0.00    0.00     0.01     0.00    33.58     0.00    3.39    3.39    0.00   2.71   0.00
dm-2              0.00     0.00    0.00    0.00     0.01     0.00    41.74     0.00    6.74    4.59  143.25   4.86   0.00
[...]
```

### free -m

`Attention:` buffer（**缓冲**）用于块设备，cache（**缓存**）用于文件系统。使用 ZFS 文件系统可能会造成一种缺少空闲内存的假象，ZFS 自己的文件系统缓存不计算在`free -m`内。

```bash
$ free -m
              total        used        free      shared  buff/cache   available
Mem:           7983        3746        1255           1        2981        3879
Swap:             0           0           0
```

### sar -n DEV 1

用于查看网络流量。

```bash
$ sar -n DEV 1
Linux 3.10.0-514.el7.x86_64 (n1.k8s.com)        03/26/2018      _x86_64_        (8 CPU)

10:26:58 AM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
10:26:59 AM vethdd9260f      0.00      0.00      0.00      0.00      0.00      0.00      0.00
10:26:59 AM vethcb3685f      6.00      4.00      0.63      0.42      0.00      0.00      0.00
10:26:59 AM vethb280b46      0.00      0.00      0.00      0.00      0.00      0.00      0.00
10:26:59 AM vethb9914b2      0.00      0.00      0.00      0.00      0.00      0.00      0.00
10:26:59 AM vetha23f71e      0.00      0.00      0.00      0.00      0.00      0.00      0.00
10:26:59 AM        lo     45.00     45.00     10.41     10.41      0.00      0.00      0.00
10:26:59 AM veth1a67b80      0.00      0.00      0.00      0.00      0.00      0.00      0.00
10:26:59 AM    ens160     89.00     88.00     10.67     16.42      0.00      0.00      0.00
10:26:59 AM flannel.1      4.00      6.00      0.37      0.55      0.00      0.00      0.00
10:26:59 AM   docker0      6.00      4.00      0.55      0.42      0.00      0.00      0.00
10:26:59 AM veth1bb7d82      0.00      0.00      0.00      0.00      0.00      0.00      0.00
[...]
```

### sar -n TCP,ETCP 1

```bash
$ sar -n TCP,ETCP 1
Linux 3.10.0-514.el7.x86_64 (n1.k8s.com)        03/26/2018      _x86_64_        (8 CPU)

10:31:09 AM  active/s passive/s    iseg/s    oseg/s
10:31:10 AM      1.00      0.00    202.00    206.00

10:31:09 AM  atmptf/s  estres/s retrans/s isegerr/s   orsts/s
10:31:10 AM      0.00      0.00      0.00      0.00      1.00

10:31:10 AM  active/s passive/s    iseg/s    oseg/s
10:31:11 AM      0.00      0.00     70.00     69.00
[...]
```

### top

```bash
$ top
top - 10:33:44 up 4 days, 19:53,  2 users,  load average: 0.30, 0.25, 0.21
Tasks: 203 total,   1 running, 202 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.7 us,  0.5 sy,  0.0 ni, 98.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  8175148 total,  1275752 free,  3840864 used,  3058532 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  3967860 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                                   
 1450 root      20   0  990376  82964  29064 S   2.3  1.0 190:36.23 kubelet                                                                                                                   
 1238 root      20   0  404012 263748  32680 S   2.0  3.2 172:04.16 kube-apiserver                                                                                                            
  978 root      20   0 10.213g 107832  13040 S   1.7  1.3 171:20.35 etcd                                                                                                                      
 1286 root      20   0  909376  60972  18820 S   1.7  0.7 153:23.73 dockerd                                                                                                                   
  712 root      20   0   54924  27172  13376 S   1.0  0.3  53:16.15 kube-scheduler                                                                                                            
 6723 systemd+  20   0 6320356 867680  16256 S   0.7 10.6  11:21.24 java                                                                                                                      
   22 root      rt   0       0      0      0 S   0.3  0.0   0:24.12 migration/3                                                                                                               
  971 root      20   0   54948  23564  13252 S   0.3  0.3  13:46.43 kube-proxy                                                                                                                
  976 ceph      20   0  400784  53644  11012 S   0.3  0.7  24:56.87 ceph-mon                                                                                                                  
 1301 root      20   0  904872  28140  13036 S   0.3  0.3  24:26.56 docker-containe                                                                                                           
 6656 systemd+  20   0 7463428 765872  16348 S   0.3  9.4   6:11.82 java                                                                                                                      
12235 root       0 -20       0      0      0 S   0.3  0.0   0:00.58 kworker/5:1H                                                                                                              
23261 root      20   0 1722644 321224   7456 S   0.3  3.9  43:57.80 fluentd                                                                                                                   
27263 root      20   0       0      0      0 S   0.3  0.0   0:11.16 kworker/1:2                                                                                                               
    1 root      20   0  191092   4164   2436 S   0.0  0.1   0:09.49 systemd                                                                                                                   
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.36 kthreadd   
[...]
```

## Tools

+ [perf_events](http://brendangregg.com/perf.html)
+ [perf-tools](https://github.com/brendangregg/perf-tools)
+ [even more ...](http://brendangregg.com/linuxperf.html)

```bash
$ ll perf-tools/bin
total 0
lrwxrwxrwx 1 root root 16 Mar 26 09:02 bitesize -> ../disk/bitesize
lrwxrwxrwx 1 root root 15 Mar 26 09:02 cachestat -> ../fs/cachestat
lrwxrwxrwx 1 root root 12 Mar 26 09:02 execsnoop -> ../execsnoop
lrwxrwxrwx 1 root root 19 Mar 26 09:02 funccount -> ../kernel/funccount
lrwxrwxrwx 1 root root 19 Mar 26 09:02 funcgraph -> ../kernel/funcgraph
lrwxrwxrwx 1 root root 20 Mar 26 09:02 funcslower -> ../kernel/funcslower
lrwxrwxrwx 1 root root 19 Mar 26 09:02 functrace -> ../kernel/functrace
lrwxrwxrwx 1 root root 12 Mar 26 09:02 iolatency -> ../iolatency
lrwxrwxrwx 1 root root 10 Mar 26 09:02 iosnoop -> ../iosnoop
lrwxrwxrwx 1 root root 12 Mar 26 09:02 killsnoop -> ../killsnoop
lrwxrwxrwx 1 root root 16 Mar 26 09:02 kprobe -> ../kernel/kprobe
lrwxrwxrwx 1 root root 12 Mar 26 09:02 opensnoop -> ../opensnoop
lrwxrwxrwx 1 root root 22 Mar 26 09:02 perf-stat-hist -> ../misc/perf-stat-hist
lrwxrwxrwx 1 root root 21 Mar 26 09:02 reset-ftrace -> ../tools/reset-ftrace
lrwxrwxrwx 1 root root 11 Mar 26 09:02 syscount -> ../syscount
lrwxrwxrwx 1 root root 17 Mar 26 09:02 tcpretrans -> ../net/tcpretrans
lrwxrwxrwx 1 root root 16 Mar 26 09:02 tpoint -> ../system/tpoint
lrwxrwxrwx 1 root root 14 Mar 26 09:02 uprobe -> ../user/uprobe
```
