---
title: K8S集群部署 - Harhor
date: 2018-02-03 16:35:11
categories: DevOps
---
## CI/CD和DevOps

![](/images/k_DevOps.png)
图片来自[CloudBees](https://www.cloudbees.com).

<!-- more -->

>[- - -摘自阮一峰老师日志](http://www.ruanyifeng.com/blog/2015/09/continuous-integration.html)

> 持续集成（Continuous Integration）的目的是让产品可以快速迭代，同时还能保持高质量。核心措施是代码集成到主干之前，必须通过自动化测试。只要有一个测试用例失败，就不能集成。优点就是能快速发现错误，防止分支大幅偏离主干。   

>持续交付（Continuous Delivery）是指频繁的将软件的新版本交付给 QA或用户 以供评审。如果评审通过，代码就进入生产阶段。持续交付可以看作持续集成的下一步。它强调的是，不管怎么更新软件是随时随地可以交付的。

>持续部署（Continuous Deployment）是持续交付的下一步，指的是代码通过评审以后，自动部署到生产环境。持续部署的目标是，代码在任何时刻都是可部署的，可以进入生产阶段。持续部署的前提是能自动化完成测试、构建、部署等步骤。
>它与持续交付的区别如下图所示：
![](/images/k_CD.png)

本[Kubernetes](https://kubernetes.io)集群部署学习参考[Jimmy Song](https://jimmysong.io)的著作。

## K8S集群部署规划

K8S集群使用二进制部署，同时开启集群TLS安全认证。

<style>
table th:nth-of-type(1) {
    width: 110px;
}
table th:nth-of-type(2) {
    width: 130px;
}
</style>

|     IP     | Hostname | Roles   |
|   :---:   |   :---:  | :---   |
| 10.0.77.16 | repo.k8s.com | Harbor（私有镜像仓库）|
| 10.0.77.17 | a1.k8s.com | master node kube-apiserver kube-controller-manager kube-scheduler kubelet kube-proxy etcd flannel
| 10.0.77.18 | a2.k8s.com | node kubectl kube-proxy flannel etcd |
| 10.0.77.19 | a3.k8s.com | node kubectl kube-proxy flannel etcd |

## Harbor

Harbor是由VMware公司开源的企业级的Docker Registry管理项目，它包括权限管理(RBAC)、LDAP、日志审核、管理界面、自我注册、镜像复制和中文支持等功能。

> Harbor的所有服务组件都是在Docker中部署的，所以官方安装使用Docker-compose快速部署，所以我们需要安装Docker、Docker-compose。由于Harbor是基于Docker Registry V2版本，所以就要求Docker版本不小于1.10.0，Docker-compose版本不小于1.6.0。

```bash
$ yum install docker

$ curl -L https://github.com/docker/compose/releases/download/1.13.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
$
```

未完待续...
