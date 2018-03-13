---
title: CephFS文件系统
date: 2018-03-13 20:34:47
categories: DevOps
---
## [CEPH简介](http://docs.ceph.org.cn/start/intro/)

不管是为云平台提供 Ceph 对象存储和/或 Ceph 块设备，还是 Ceph 文件系统或者它用，所有 Ceph 存储集群的部署都始于部署一个个 Ceph 节点、网络和 Ceph 存储集群。 Ceph 存储集群至少需要**`一个 Ceph Monitor 和两个 OSD 守护进程`**。**而运行 Ceph 文件系统客户端时，则必须要有元数据服务器。**

MDSs（ Metadata Server ）: Ceph 元数据服务器（ MDS ）为 Ceph 文件系统存储元数据（也就是说，Ceph 块设备和 Ceph 对象存储不使用 MDS ）。元数据服务器使得 POSIX 文件系统的用户可以在不对 Ceph 存储集群造成负担的前提下，执行诸如 ls、find 等基本命令。

MDS也可以在多节点部署实现冗余。MDS守护进程可以被配置为活跃或者被动状态，活跃的MDS为主MDS，其他的MDS处于备用状态，当主MDS节点故障时，备用MDS节点会接管其工作并被提升为主节点。

<!-- more -->

## 部署MDS

[**Important:**](http://docs.ceph.org.cn/rados/deployment/ceph-deploy-mds/) 必须部署至少一个元数据服务器才能使用 CephFS 文件系统，多个元数据服务器并行运行仍处于实验阶段。不要在生产环境下运行多个元数据服务器。

接 [快速部署 Ceph 集群](https://acquaai.github.io/2018/03/13/ceph-cluster/) 学习环境继续。

在 ceph 管理节点上执行如下命令为集群增加一个MDS: 

```
ceph-deploy mds create {host-name}[:{daemon-name}] [{host-name}[:{daemon-name}] ...]
```

```
$ ceph-deploy --overwrite-conf mds create n1.k8s.com
```

## 创建 CephFS

CephFS需要使用两个Pool来分别存储数据和元数据，分别创建fs_data和fs_metadata两个Pool。

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

## Mount CephFS

客户端访问Ceph FS有两种方式：

+ Kernel内核驱动：Linux内核从2.6.34版本开始加入对CephFS的原声支持
+ Ceph FS FUSE： FUSE即Filesystem in Userspace

### 用内核驱动挂载 CEPH 文件系统

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

+ [CEPH FS QUICK START](http://docs.ceph.com/docs/master/start/quick-cephfs/#)
