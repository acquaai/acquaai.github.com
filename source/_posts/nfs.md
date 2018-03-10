---
title: NFS Share 
date: 2018-03-10 11:27:25
categories: DevOps
---
## Server

```bash
$ setsebool -P virt_use_nfs 1 (OR SELinux status: disabled)
$ yum -y install nfs-utils libnfsidmap
$ systemctl enable rpcbind nfs-server
$ systemctl start rpcbind nfs-server rpc-statd nfs-idmapd

$ cat /etc/expots
/nfs/mysql57/mysql_data 10.0.77.0/27(rw,sync,anonuid=1000,anongid=1000,no_root_squash)

$ chmod -R 755 /nfs
$ exportfs -r
```

> + rw: 读写
> + sync: 文件同时写入硬盘和内存
> + anonuid: 匿名用户的UID值
> + anongid: 匿名用户的GID值。备注：其中anonuid=1000,anongid=1000,为此目录用户web的ID号，达到连接NFS用户权限一致
> + no_root_squash: NFS客户端连接服务端时如果使用的是root的话，那么对服务端分享的目录来说，也拥有root权限，开启这项是不安全的

<!-- more -->

## Client

```bash
$ yum -y install nfs-utils
$ systemctl start rpcbind.service

$ showmount -e 10.0.77.16
Export list for 10.0.77.16:
/nfs/mysql57/mysql_data 10.0.77.0/27

$ mount -t nfs 10.0.77.16:/nfs/mysql57/mysql_data /nfs

#NFS默认是用UDP协议，使用TCP协议传输更稳定
$ mount -t nfs 10.0.77.16:/nfs/mysql57/mysql_data /nfs -o proto=tcp -o nolock

#开机自动挂载
$ cat /etc/fstab  
10.0.77.16:/nfs/mysql57/mysql_data    /nfs    nfs4 rw,hard,intr,proto=tcp,port=2049,noauto    0  0
```
