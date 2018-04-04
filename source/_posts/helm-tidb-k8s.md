---
title: Helm deploy TiDB on Kubernetes
date: 2018-04-04 18:24:11
categories: DevOps
---
# tl;dr
 
本文将使用 `Helm` 包管理器在 `Kubernetes` 集群上部署 `TiDB`。

[Helm on Kubernetes](https://acquaai.github.io/2018/04/02/helm/)

<!-- more -->

## TiDB Chart

> [TiDB](https://pingcap.com) 是新一代开源分布式 NewSQL 数据库，模型受 Google Spanner / F1 论文的启发，实现了自动的水平伸缩，强一致性的分布式事务，基于 Raft 算法的多副本复制等重要 NewSQL 特性。TiDB 结合了 RDBMS 和 NoSQL 的优点，部署简单，在线弹性扩容和异步表结构变更不影响业务， 真正的异地多活及自动故障恢复保障数据安全，同时兼容 MySQL 协议，使迁移使用成本降到极低。

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

Chart 的文件在[这里](https://github.com/acquaai/charts/tree/master/incubator/tidb)。

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
$ helm install --name pingcap ./tidb
NAME:   pingcap
LAST DEPLOYED: Wed Apr  4 17:20:31 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/StorageClass
NAME                 PROVISIONER        AGE
ceph-tidb (default)  kubernetes.io/rbd  1s

==> v1/Service
NAME             TYPE       CLUSTER-IP  EXTERNAL-IP  PORT(S)             AGE
pingcap-tidb-pd  ClusterIP  None        <none>       2379/TCP,2380/TCP   1s
pingcap-tidb-db  ClusterIP  None        <none>       4000/TCP,10080/TCP  1s

==> v1beta1/Deployment
NAME             DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
pingcap-tidb-db  1        1        1           0          0s

==> v1beta1/StatefulSet
NAME             DESIRED  CURRENT  AGE
pingcap-tidb-pd  3        1        0s
pingcap-tidb-kv  3        1        0s

==> v1/Pod(related)
NAME                              READY  STATUS             RESTARTS  AGE
pingcap-tidb-db-846cdff8c6-z879m  0/1    ContainerCreating  0         0s
pingcap-tidb-pd-0                 0/1    ContainerCreating  0         0s
pingcap-tidb-kv-0                 0/1    Init:0/1           0         0s

$ helm status pingcap
```

> 注意`Chart.yaml`文件中 **name 和 version** 的值及`--name` **release name**，否则会报错：
> 
> Error: release PingCAP failed: Service "PingCAP-TiDB-pd" is invalid: metadata.name: Invalid value: "PingCAP-TiDB-pd": a DNS-1035 label must consist of lower case alphanumeric characters or '-', start with an alphabetic character, and end with an alphanumeric character (e.g. 'my-name',  or 'abc-123', regex used for validation is '[a-z]([-a-z0-9]*[a-z0-9])?')
> 

```bash
$ helm list
NAME    REVISION        UPDATED                         STATUS          CHART           NAMESPACE
pingcap 1               Wed Apr  4 17:20:31 2018        DEPLOYED        tidb-v2.0.0     default

$ helm delete pingcap
release "pingcap" deleted
```

```bash
$ helm install --name pingcap ./tidb
Error: a release named pingcap already exists.
Run: helm ls --all pingcap; to check the status of the release
Or run: helm del --purge pingcap; to delete it

$ helm ls --all
NAME                    REVISION        UPDATED                         STATUS  CHART                   NAMESPACE
pingcap                 1               Tue Apr  4 17:20:31 2018        DELETED tidb-v2.0.0             default  

$ helm del --purge pingcap
release "pingcap" deleted
```

### Deploy Chart into Artifactory

```json
$ helm package ./tidb
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

> --max-open-files=1000000: Number of files that can be opened by Kubelet process. [default=1000000]

coming soon...


**References**

+ [TiDB快速部署](https://pingcap.com/docs-cn/op-guide/docker-compose/#自定义集群)
+ [PingCAP ∙ GitHub](https://github.com/pingcap)
+ [TiDB FAQ](https://github.com/pingcap/docs/blob/master/FAQ.md)
+ [Running TiDB on Kubernetes](https://banzaicloud.com/blog/tidb-kubernetes/)
+ [banzaicloud/banzai-charts](https://github.com/banzaicloud/banzai-charts/tree/master/incubator/tidb)
