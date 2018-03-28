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
/nfs/sonar/conf 10.0.77.0/27(rw,sync,no_root_squash)
/nfs/sonar/data 10.0.77.0/27(rw,sync,no_root_squash)
/nfs/sonar/extensions 10.0.77.0/27(rw,sync,no_root_squash)
/nfs/sonar/logs 10.0.77.0/27(rw,sync,no_root_squash)

$ chmod -R 777 /nfs
$ exportfs -r
```

<!-- more -->

> + rw: 该主机对该共享目录有读写权限
> + sync: 资料同步写入到内存与硬盘中
> + no_root_squash: 客户机用root访问该共享文件夹时，不映射root用户
> + root_squash: 客户机用root用户访问该共享文件夹时，将root用户映射成匿名用户
> + no_subtree_check: 不检查父目录权限
> + subtree_check: 如果共享/usr/bin之类的子目录时，强制NFS检查父目录的权限（默认）
> + anonuid: 将客户机上的用户映射成指定的本地用户ID的用户
> + anongid: 将客户机上的用户映射成属于指定的本地用户组ID
> + secure: NFS通过1024以下的安全TCP/IP端口发送
> + insecure: NFS通过1024以上的端口发送
> + wdelay: 如果多个用户要写入NFS目录，则归组写入（默认）
> + no_wdelay: 如果多个用户要写入NFS目录，则立即写入，当使用async时，无需此设置

## Client

```bash
$ yum -y install nfs-utils
$ systemctl start rpcbind.service

$ showmount -e 10.0.77.16
Export list for 10.0.77.16:
/nfs/sonar/logs       10.0.77.0/27
/nfs/sonar/extensions 10.0.77.0/27
/nfs/sonar/data       10.0.77.0/27
/nfs/sonar/conf       10.0.77.0/27

$ mount -t nfs 10.0.77.16:/nfs/sonar/logs /nfs

#NFS默认是用UDP协议，使用TCP协议传输更稳定
$ mount -t nfs 10.0.77.16:/nfs/sonar/logs /nfs -o proto=tcp -o nolock

#开机自动挂载
$ cat /etc/fstab  
10.0.77.16:/nfs/sonar/logs    /nfs    nfs4 rw,hard,intr,proto=tcp,port=2049,noauto    0  0
```
