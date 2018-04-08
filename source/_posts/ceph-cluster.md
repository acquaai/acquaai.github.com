---
title: Ceph Distributed Storage System
date: 2018-03-13 17:18:28
categories: DevOps
---
## Ceph 介绍

Ceph 是一个符合 POSIX 的开源分布式存储系统，能提供 Ceph Object、Ceph RBD 和 Ceph FS 的存储能力。Ceph Cluster至少需要一个`Ceph Monitor` 和两个`OSD Daemon`，当运行 Ceph 文件系统客户端时，还必须要有元数据服务器（MDS）。

Components:

+ Monitors: 基于 PAXOS 算法维护集群状态的各种图表，包括监视器图、OSD图、PG图、CRUSH图。Ceph 保存( `epoch` )发生在 Monitor、OSD 和 PG 上的每一次状态变更的历史信息。
+ OSDs: 存储数据，处理数据的复制、恢复、回填、再均衡，并通过检查其它 OSD 守护进程的心跳来向 Monitors 提供监控信息。当存储集群设定为 2个副本 时，至少需要 2个OSD守护进程，集群才能达到` active + clean `的状态，Ceph 默认为 3副本。
+ MDSs: 存储 Ceph FS 元数据。（针对 Object/RBD 存储不需要 MDS）MDS也可以在多节点部署实现冗余。MDS守护进程可以被配置为活跃或者被动状态，活跃的MDS为主MDS，其他的MDS处于备用状态，当主MDS节点故障时，备用MDS节点会接管其工作并被提升为主节点。

<!-- more -->

![](/images/cephstack.png)

![](/images/cepharch.png)

**RADOS**

RADOS (Reliable Autonomic Distributed Object Store)自身就是一套完整的对象存储系统，包括 Ceph 的基础服务（Monitor、OSD、MDS），存储应用数据，同时提供 Ceph 的高可用性、高扩展性和高自动化特性。

**LibRados**

对 RADOS 进行抽象和封装，向上层提供不同的 API（RGW、RBD、FS）。提供 C/C++ 原生 API。

**APP的接口**

抽象 LibRados，便于应用和客户端使用。

+ RGW: 提供与 S3 和 Swift 兼容的 RESTful API 的 gateway。通过 RGW 可以将 RADOS 响应转化为 HTTP 响应，反之亦然。
+ RBD: 提供一个标准的块设备接口服务。
+ CephFS: 提供一个 POSIX 兼容的分布式文件系统。

**Client**

Ceph 应用接口的应用方式，如基于 LibRados 直接开发的对象存储应用，基于 RGW 开发的对象存储应用，基于RBD实现的云硬盘等。

一个 Ceph Cluster 逻辑上可以划分为多个 Pool，一个 Pool 由若干个逻辑 PG 组成。

一个文件会被切分为多个 Object，每个 Object 被映射到一个 PG，每个 PG 会根据 CRUSH 算法映射到一组 OSD（根据副本数），其中第一个 OSD（Primary OSD）为主，其它为备，OSD之间通过心跳来互相监控存活状态。一般来讲增加 PG 的数量能降低 OSD 负载，每个OSD大约分配 50～100 PG。

CRUSH: 是 Ceph 使用的数据分布算法，类似一致性哈希，使数据分配到预期的地方。

## 部署 Ceph 集群

IP  | Hostname | Roles |
--- | ---| ---: |
10.0.77.16    |  repo.k8s.com  |  admin-node, deploy-node
10.0.77.17    |  n1.k8s.com  |   mon
10.0.77.18    |  n2.k8s.com  |  osd.1
10.0.77.19    |  n3.k8s.com  |  osd.2

### Ceph yum 源

```
[root@repo ~]# cat /etc/yum.repos.d/ceph.repo

[ceph-noarch]
name=Ceph noarch packages
baseurl=https://download.ceph.com/rpm-jewel/el7/noarch
enabled=1
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc
```

### 安装 ceph-deploy

```bash
[root@repo ~]# yum update -y && yum -y install ceph-deploy
```

### 配置 Ceph 用户

`every node`新建 ceph 集群 k8sceph 用户，并确保该用户具有sudo权限。

```bash
$ useradd -d /home/k8sceph -m k8sceph
$ passwd k8sceph

$ echo "k8sceph ALL = (root) NOPASSWD:ALL" | tee /etc/sudoers.d/k8sceph
$ chmod 0440 /etc/sudoers.d/k8sceph

#禁用 TTY
在 CentOS 和 RHEL 上执行 ceph-deploy 命令时可能会报错。如果你的 Ceph 节点默认设置了 requiretty ，执行 sudo visudo 禁用它，并找到 Defaults requiretty 选项，把它改为 Defaults:ceph !requiretty 或者直接注释掉，这样 ceph-deploy 就可以用之前创建的用户连接了。

$ visudo -f /etc/sudoers

Defaults:k8sceph     !requiretty

#优先级/首选项
$ yum -y install yum-plugin-priorities
```

### 配置 SSH 免密登录

```bash
[root@repo ~]# ssh-keygen
[root@repo ~]# ssh-copy-id k8sceph@n1.k8s.com
[root@repo ~]# ssh-copy-id k8sceph@n2.k8s.com
[root@repo ~]# ssh-copy-id k8sceph@n3.k8s.com

#修改 repo.k8s.com 控制节点上 ~/.ssh/config 文件，设置当不指定用户时登录到 n1~n3 的用户为 k8sceph.
[root@repo ~]# vi ~/.ssh/config
Host c1
   Hostname c1
   User sdsceph
Host c2
   Hostname c2
   User sdsceph
Host c3
   Hostname c3
   User sdsceph

#repo.k8s.com 控制节点上 ssh 无密登录到 n1~n3 的测试
[root@repo ~]# ssh n1.k8s.com
```

### 配置 NTP 服务

`every node:`

```bash
$ ntpstat 
synchronised to NTP server (192.168.0.118) at stratum 4 
   time correct to within 56 ms
   polling server every 1024 s
```

## 安装 Ceph

```bash
[root@repo ceph-cluster]# mkdir ceph-cluster && cd ceph-cluster
```

在管理节点上，进入刚创建的放置配置文件的目录，用 ceph-deploy 执行如下步骤。

```bash
[root@repo ceph-cluster]# ceph-deploy new n1.k8s.com

#生成下面3个文件：
[root@repo ceph-cluster]# ls
ceph.conf  ceph-deploy-ceph.log  ceph.mon.keyring
```

把Ceph 配置文件里的ceph.conf里的默认副本数从 3 改成 2， 这样只有两个 OSD 也可以达到 active + clean 状态。把下面这行加入 [global] 段：

`osd pool default size = 2`

从管理节点执行各节点安装ceph：

```bash
[root@repo ceph-cluster]# ceph-deploy install repo.k8s.com n1.k8s.com n2.k8s.com n3.k8s.com

#各个节点安装完后显示：
Complete!
Running command: sudo ceph --version
ceph version 10.2.10 (5dc1e4c05cb68dbf62ae6fce3f0700e4654fdbbe)
```

ceph-deploy 将在各节点安装 Ceph 。 注：如果你执行过 ceph-deploy purge ，你必须重新执行这一步来安装 Ceph 。

配置初始 monitor(s)、并收集所有密钥，这里只有 n1：
> 没有识别完整的主机名 n1.k8s.com，无赖在/etc/hosts文中增加了 n1。

```
[root@repo ceph-cluster]# ceph-deploy mon create-initial
```

完成上述操作后，当前目录里应该会出现这些密钥环：

+ ceph.bootstrap-mds.keyring
+ ceph.bootstrap-mgr.keyring
+ ceph.bootstrap-osd.keyring
+ ceph.bootstrap-rgw.keyring
+ ceph.client.admin.keyring

```
[root@repo ceph-cluster]# ll
total 268
-rw------- 1 root root    113 Mar 13 16:02 ceph.bootstrap-mds.keyring
-rw------- 1 root root     71 Mar 13 16:02 ceph.bootstrap-mgr.keyring
-rw------- 1 root root    113 Mar 13 16:02 ceph.bootstrap-osd.keyring
-rw------- 1 root root    113 Mar 13 16:02 ceph.bootstrap-rgw.keyring
-rw------- 1 root root    129 Mar 13 16:02 ceph.client.admin.keyring
-rw-r--r-- 1 root root    215 Mar 13 15:26 ceph.conf
-rw-r--r-- 1 root root 245513 Mar 13 16:02 ceph-deploy-ceph.log
-rw------- 1 root root     73 Mar 13 15:23 ceph.mon.keyring
```

> 只有在安装 Hammer 或更高版时才会创建 bootstrap-rgw 密钥环。
>  如果此步失败并输出类似于如下信息 “Unable to find /etc/ceph/ceph.client.admin.keyring”，请确认 ceph.conf 中为 monitor 指定的 IP 是 Public IP，而不是 Private IP。

n1.k8s.com上看到ceph monitor进程已经启动:

```bash
[root@n1 ~]# ps -ef|grep ceph-mon|grep -v grep
ceph     17915     1  0 15:58 ?        00:00:00 /usr/bin/ceph-mon -f --cluster ceph --id n1 --setuser ceph --setgroup ceph
```

接下来添加两个OSD到n2和n3。OSD是存储数据的节点，实际上需要为OSD提供独立的存储空间，如一个独立的磁盘。为 OSD 及其日志使用独立硬盘或分区，请参考[ceph-deploy osd](http://docs.ceph.org.cn/rados/deployment/ceph-deploy-osd)。

本学习使用系统本地磁盘创建目录提供给OSD使用。

```bash
[root@repo ceph-cluster]# ssh n2.k8s.com
[k8sceph@n2 ~]$ sudo mkdir -p /ceph/osd0
[k8sceph@n2 ~]$ sudo chown -R ceph:ceph /ceph/osd0
[k8sceph@n2 ~]$ exit

[root@repo ceph-cluster]# ssh n3.k8s.com
[k8sceph@n3 ~]$ sudo mkdir -p /ceph/osd1
[k8sceph@n3 ~]$ sudo chown -R ceph:ceph /ceph/osd1
[k8sceph@n3 ~]$ exit
```

从管理节点执行 ceph-deploy 来准备 OSD :

```bash
[root@repo ceph-cluster]# ceph-deploy osd prepare n2.k8s.com:/ceph/osd0 n3.k8s.com:/ceph/osd1
```

激活 OSD :

```bash
[root@repo ceph-cluster]# ceph-deploy osd activate n2.k8s.com:/ceph/osd0 n3.k8s.com:/ceph/osd1
```

用 ceph-deploy 把配置文件和 admin 密钥拷贝到管理节点和 Ceph 节点，这样你每次执行 Ceph 命令行时就无需指定 monitor 地址和 ceph.client.admin.keyring 了。

```
[root@repo ceph-cluster]# chmod +r /etc/ceph/ceph.client.admin.keyring
[root@repo ceph-cluster]# ceph-deploy admin repo.k8s.com n1.k8s.com n2.k8s.com n3.k8s.com
```

> ceph-deploy 和本地管理主机（ admin-node ）通信时，必须通过主机名可达。必要时可修改 /etc/hosts ，加入管理主机的名字。

确保对 ceph.client.admin.keyring 有正确的操作权限:
$ sudo chmod +r /etc/ceph/ceph.client.admin.keyring

查看ceph集群中OSD节点的状态：

```bash
[root@repo ceph-cluster]# ceph osd tree
ID WEIGHT  TYPE NAME     UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-1 0.35339 root default                                    
-2 0.17670     host n2                                     
 0 0.17670         osd.0      up  1.00000          1.00000 
-3 0.17670     host n3                                     
 1 0.17670         osd.1      up  1.00000          1.00000
```

查看集群健康状态:

```bash
[root@repo ceph-cluster]# ceph health
HEALTH_OK
```

如果在某些地方碰到麻烦，想从头再来，

```
可以用下列命令清除配置：
ceph-deploy purgedata {ceph-node} [{ceph-node}]
ceph-deploy forgetkeys

用下列命令可以连 Ceph 安装包一起清除：
ceph-deploy purge {ceph-node} [{ceph-node}]
```

## 操作集群

### 对象存储测试

要把对象存入 Ceph 存储集群，客户端必须做到：

  + 指定对象名
  + 指定存储池

查看集群中的存储池：

```bash
[root@repo ceph-cluster]# ceph osd lspools
0 rbd,
```

集群上只有rbd一个存储池，创建名为 k8s 的存储池: 

```bash
[root@repo ceph-cluster]# ceph osd pool create k8s 100   # 100个 PG
pool 'k8s' created
```

先创建一个对象，用 rados put 命令加上对象名、一个有数据的测试文件路径、并指定存储池: 

```bash
[root@repo ~]# echo {Test-data} > testfile.txt

rados put {object-name} {file-path} --pool={pool-name}

[root@repo ~]# rados put test-object-1 testfile.txt --pool=k8s
```

确认 Ceph 存储集群存储了此对象: 

```bash
[root@repo ~]# rados -p k8s ls
test-object-1
```

定位对象: 

```bash
ceph osd map {pool-name} {object-name}

[root@repo ~]# ceph osd map k8s test-object-1
osdmap e12 pool 'k8s' (1) object 'test-object-1' -> pg 1.74dc35e2 (1.62) -> up ([0,1], p0) acting ([0,1], p0)
```

删除对象: 

```bash
[root@repo ~]# rados rm test-object-1 --pool=k8s
```

### 文件系统测试

[**Important:**](http://docs.ceph.org.cn/rados/deployment/ceph-deploy-mds/) 必须部署至少一个元数据服务器才能使用 CephFS 文件系统，多个元数据服务器并行运行仍处于实验阶段，不要在生产环境下运行多个元数据服务器。

在 Ceph 管理节点上执行如下命令为集群增加一个MDS:

```bash
ceph-deploy mds create {host-name}[:{daemon-name}] [{host-name}[:{daemon-name}] ...]
```

```bash
$ ceph-deploy --overwrite-conf mds create n1.k8s.com
```

1.创建 CephFS

CephFS需要使用两个Pool来分别存储数据和元数据，分别创建fs_data和fs_metadata两个Pool。

> PG 数量指定一般遵循的公式：

> + 集群 PG 总数 = ( OSD总数 * 100 ) / 数据最大副本数
> + 单个存储池 PG 数 = ( OSD总数 * 100 ) / 数据最大副本数 / 存储池数

PG 的最终结果以最接近上述公式的 2^N^ 向上取值，如：
PG = 2 * 100 / 2 / 2 = 50，向上取 2^6^ 为 64，最接近 50，故 `PG = 64`

```bash
$ ceph osd pool create fs_data 128
pool 'fs_data' created

$ ceph osd pool create fs_metadata 128
pool 'fs_metadata' created

$ ceph osd lspools
0 rbd,1 k8s,2 fs_data,3 fs_metadata,
```

```bash
$ ceph fs new cephfs fs_metadata fs_data
new fs with metadata pool 3 and data pool 2

$ ceph fs ls
name: cephfs, metadata pool: fs_metadata, data pools: [fs_data ]
```

2.Mount CephFS

客户端访问Ceph FS有两种方式：

+ Kernel内核驱动：Linux内核从2.6.34版本开始加入对CephFS的原声支持
+ Ceph FS FUSE： FUSE即Filesystem in Userspace

3.用内核驱动挂载 Ceph 文件系统

```bash
[root@mysql01 ~]$ uname -r
3.10.0-693.5.2.el7.x86_64

[root@mysql01 ~]$ mkdir /mnt/cephfs
[root@mysql01 ~]$ mkdir /etc/ceph

[root@mysql01 ~]$ scp -P 8086 root@10.0.77.16:/root/ceph-cluster/ceph.client.admin.keyring /etc/ceph

[root@mysql01 ceph]$ mv ceph.client.admin.keyring admin.secret
[root@mysql01 ceph]$ cat admin.secret
AQCwhKdaIbKRHxAAYfCmoj+VRsfQXSs0784Kow==

[root@mysql01 ~]$ mount -t ceph 10.0.77.17:6789,10.0.77.18:6789,10.0.77.19:6789:/ /mnt/cephfs -o name=admin,secretfile=admin.secret

Filesystem                                         Size  Used Avail Use% Mounted on
10.0.77.17:6789,10.0.77.18:6789,10.0.77.19:6789:/  362G   30G  333G   9% /mnt/cephfs
```

> `secretfile= option`时，CentOS7.3 - 3.10.0-693.5.2.el7.x86_64有bug，CentOS7.4 - 3.10.0-514.el7.x86_64时正常，直接使用`secret= option`时都正常。

**Reference**

+ [Ceph Documentation](http://docs.ceph.org.cn/start/)
+ [CEPH FS QUICK START](http://docs.ceph.com/docs/master/start/quick-cephfs/#)
