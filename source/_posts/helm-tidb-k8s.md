---
title: Helm deploy TiDB on Kubernetes
date: 2018-04-04 18:24:11
categories: DevOps
---
# tl;dr
 
本文将使用 `Helm` 包管理器在 `Kubernetes` 集群上部署 `TiDB`。

## TiDB Chart

> [TiDB](https://pingcap.com) 是新一代开源分布式 NewSQL 数据库，模型受 Google Spanner / F1 论文的启发，实现了自动的水平伸缩，强一致性的分布式事务，基于 Raft 算法的多副本复制等重要 NewSQL 特性。TiDB 结合了 RDBMS 和 NoSQL 的优点，部署简单，在线弹性扩容和异步表结构变更不影响业务， 真正的异地多活及自动故障恢复保障数据安全，同时兼容 MySQL 协议，使迁移使用成本降到极低。

<!-- more -->

### Create [Chart](https://docs.helm.sh/developing_charts/#charts)

```bash
$ helm create tidb
Creating tidb

$ yum -y install tree
$ tree tidb
tidb
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── ingress.yaml
│   ├── NOTES.txt
│   └── service.yaml
└── values.yaml

2 directories, 7 files
```

+ charts: 本 TiDB Chart 依赖的 chart，当前为空
+ Chart.yaml: 描述 Chart 的基本信息
+ templates: k8s manifest的模板文件目录。模板使用 chart 配置的值生成 manifest 文件，遵守 Go 语言的语法。
+ templates/NOTES.txt: 描述 chart 的使用说明
+ values.yaml: chart 配置的默认值

### Config TiDB Chart

创建 TiKV 的存储类，TiKV 动态请求 PVC。

```bash
$ kubectl create secret generic ceph-secret --type="kubernetes.io/rbd" \
  --from-literal=key='QVFDd2hLZGFJYktSSHhBQVlmQ21vaitWUnNmUVhTczA3ODRLb3c9PQ=='

$ kubectl create -f tikv-storageclass.yaml  
```

Chart 的文件在[这里](https://github.com/acquaai/K8S/tree/master/charts/incubator/tidb)。

### Test TiDB Chart

```bash
$ helm install --dry-run --debug tidb
[debug] Created tunnel using local port: '38158'

[debug] SERVER: "127.0.0.1:38158"

[debug] Original chart version: ""
[debug] CHART PATH: /root/tidb

NAME:   invisible-pika
REVISION: 1
RELEASED: Wed Apr  4 17:19:31 2018
CHART: tidb-v2.0.0
...
```

### Install Chart

```bash
$ helm install --name pingcap tidb
NAME:   pingcap
LAST DEPLOYED: Sat Apr  7 15:57:13 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1beta1/Deployment
NAME                 DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
pingcap-tidb-db      1        1        1           0          1s
pingcap-tidb-vision  1        1        1           0          1s

==> v1beta1/StatefulSet
NAME             DESIRED  CURRENT  AGE
pingcap-tidb-pd  3        1        1s
pingcap-tidb-kv  3        1        1s

==> v1/Pod(related)
NAME                                  READY  STATUS             RESTARTS  AGE
pingcap-tidb-db-846cdff8c6-frb22      0/1    ContainerCreating  0         1s
pingcap-tidb-vision-76bd48b96f-pmqnl  0/1    ContainerCreating  0         1s
pingcap-tidb-pd-0                     0/1    ContainerCreating  0         1s
pingcap-tidb-kv-0                     0/1    Init:0/1           0         1s

==> v1/StorageClass
NAME                 PROVISIONER        AGE
ceph-tidb (default)  kubernetes.io/rbd  1s

==> v1/Service
NAME                 TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)                         AGE
pingcap-tidb-pd      ClusterIP  None           <none>       2379/TCP,2380/TCP               1s
pingcap-tidb-db      NodePort   10.254.164.24  <none>       4000:31975/TCP,10080:30608/TCP  1s
pingcap-tidb-vision  NodePort   10.254.30.87   <none>       8010:30895/TCP                  1s
```

```bash
$ helm status pingcap
LAST DEPLOYED: Sat Apr  7 15:57:13 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Pod(related)
NAME                                  READY  STATUS   RESTARTS  AGE
pingcap-tidb-db-846cdff8c6-frb22      1/1    Running  0         5m
pingcap-tidb-vision-76bd48b96f-pmqnl  1/1    Running  0         5m
pingcap-tidb-pd-0                     1/1    Running  0         5m
pingcap-tidb-pd-1                     1/1    Running  1         5m
pingcap-tidb-pd-2                     1/1    Running  1         5m
pingcap-tidb-kv-0                     1/1    Running  1         5m
pingcap-tidb-kv-1                     1/1    Running  0         4m
pingcap-tidb-kv-2                     1/1    Running  0         3m

==> v1/StorageClass
NAME                 PROVISIONER        AGE
ceph-tidb (default)  kubernetes.io/rbd  5m

==> v1/Service
NAME                 TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)                         AGE
pingcap-tidb-pd      ClusterIP  None           <none>       2379/TCP,2380/TCP               5m
pingcap-tidb-db      NodePort   10.254.164.24  <none>       4000:31975/TCP,10080:30608/TCP  5m
pingcap-tidb-vision  NodePort   10.254.30.87   <none>       8010:30895/TCP                  5m

==> v1beta1/Deployment
NAME                 DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
pingcap-tidb-db      1        1        1           1          5m
pingcap-tidb-vision  1        1        1           1          5m

==> v1beta1/StatefulSet
NAME             DESIRED  CURRENT  AGE
pingcap-tidb-pd  3        3        5m
pingcap-tidb-kv  3        3        5m
```

> 注意`Chart.yaml`文件中 **name 和 version** 的值及`--name` **release name**，否则会报错：
> 
> Error: release PingCAP failed: Service "PingCAP-TiDB-pd" is invalid: metadata.name: Invalid value: "PingCAP-TiDB-pd": a DNS-1035 label must consist of lower case alphanumeric characters or '-', start with an alphabetic character, and end with an alphanumeric character (e.g. 'my-name',  or 'abc-123', regex used for validation is '[a-z]([-a-z0-9]*[a-z0-9])?')

```sql
$ mysql -h 10.0.77.17 -P 31975 -u root -D test
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.10-TiDB-v2.0.0-rc.3 MySQL Community Server (Apache License 2.0)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| INFORMATION_SCHEMA |
| PERFORMANCE_SCHEMA |
| mysql              |
| test               |
+--------------------+
4 rows in set (0.01 sec)
```

```bash
$ helm list
NAME    REVISION        UPDATED                         STATUS          CHART           NAMESPACE
pingcap 1               Sat Apr  7 15:57:13 2018        DEPLOYED        tidb-v2.0.0     default  

$ helm delete pingcap
release "pingcap" deleted
```

```bash
$ helm install --name pingcap tidb
Error: a release named pingcap already exists.
Run: helm ls --all pingcap; to check the status of the release
Or run: helm del --purge pingcap; to delete it

$ helm ls --all
NAME                    REVISION        UPDATED                         STATUS  CHART                   NAMESPACE
pingcap                 1               Tue Apr  7 17:20:31 2018        DELETED tidb-v2.0.0             default  

$ helm del --purge pingcap
release "pingcap" deleted
```

### Deploy Chart into Artifactory

```json
$ helm package tidb
Successfully packaged chart and saved it to: /root/tidb-v2.0.0.tgz

$ curl -uadmin:AP412AQbN3DGPWuWXvrXvJrvzPX -T tidb-v2.0.0.tgz "http://10.0.77.17:30809/artifactory/helm-virtual/tidb-v2.0.0.tgz"
{
  "repo" : "helm-local",
  "path" : "/tidb-v2.0.0.tgz",
  "created" : "2018-04-04T09:54:26.368Z",
  "createdBy" : "admin",
  "downloadUri" : "http://10.0.77.17:30809/artifactory/helm-local/tidb-v2.0.0.tgz",
  "mimeType" : "application/x-gzip",
  "size" : "3176",
  "checksums" : {
    "sha1" : "c43f6fef95ce9c6fec22f7856cb28f1630499f13",
    "md5" : "33e0d37e351587f61124c04aeb8b87a0",
    "sha256" : "bacb3d6001794b09d554d57b6fb0e8b00acd00bf4ef1af0d0a120583dd9c4704"
  },
  "originalChecksums" : {
    "sha256" : "bacb3d6001794b09d554d57b6fb0e8b00acd00bf4ef1af0d0a120583dd9c4704"
  },
  "uri" : "http://10.0.77.17:30809/artifactory/helm-local/tidb-v2.0.0.tgz"
}
```

### QA

> 2018/04/04 03:10:07.591 tikv-server.rs:135: [ERROR] Limit("the maximum number of open file descriptors is too small, got 65536, expect greater or equal to 82920")

Linux 有文件句柄限制，默认1024，系统总限制可以查看：

```bash
$ cat /proc/sys/fs/file-max
807856
```

系统当前使用的文件句柄数量：

```bash
$ cat /proc/sys/fs/file-nr 
3424    0       807856
```

查看进程开启的句柄：

```bash
$ lsof -n |awk '{print $2}'|sort|uniq -c |sort -nr|more
```

> 修改`Docker`(/usr/lib/systemd/system/docker.service)的服务并重启：

~~LimitNOFILE=infinity~~

```bash
$ ps -ef|grep docker
root      3400  3390  1 07:56 ?        00:00:00 docker-containerd --config /var/run/docker/containerd/containerd.toml
...

$ cat /proc/3400/limits |grep "Max open files"
Max open files            65536                65536                files
```

**`LimitNOFILE=1000000`**

```bash
$ cat /proc/1393/limits |grep "Max open files"
Max open files            1000000              1000000              files
```

~~DOCKER_NOFILE=1000000 在我的CentOS Linux release 7.3.1611 (Core)中未生效。~~

**option**
limits.conf 文件实际是 Linux PAM（插入式认证模块，Pluggable Authentication Modules）中 pam_limits.so 的配置文件，突破系统的默认限制，对系统访问资源有一定保护作用。limits.conf 和 sysctl.conf 区别在于 limits.conf 是针对用户，而 sysctl.conf 是针对整个系统参数配置。

```bash
$ cat /etc/security/limits.conf
root     -     nofile     1000000
```

**References**

+ [TiDB快速部署](https://pingcap.com/docs-cn/op-guide/docker-compose/#自定义集群)
+ [PingCAP ∙ GitHub](https://github.com/pingcap)
+ [TiDB FAQ](https://github.com/pingcap/docs/blob/master/FAQ.md)
+ [Running TiDB on Kubernetes](https://banzaicloud.com/blog/tidb-kubernetes/)
+ [banzaicloud/banzai-charts](https://github.com/banzaicloud/banzai-charts/tree/master/incubator/tidb)
