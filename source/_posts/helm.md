---
title: Helm on Kubernetes
date: 2018-04-02 20:25:47
categories: DevOps
---
## Helm

[Helm](https://helm.sh) 作为 Kubernetes 的包管理平台，类似 CentOS 的 yum 或 Ubuntu 的 apt-get 来部署管理 k8s 的应用。Helm Charts 能够帮你定义、安装、升级最复杂的 Kubernetes 应用集合，它容易创建，做版本化，共享和发布。在K8s上部署一些通用应用（组件）时，利用 Helm 可以获得业界统一的`最佳实践`的配置，从而减少时间成本。

Helm Dictionary (概念):

+ chart: a package; bundle of Kubernetes [resources](https://github.com/kubernetes/charts) (能够描述最复杂的应用，提供可重复，幂等性的安装，以及提供统一的认证中心服务。)
+ Release: a chart instance is loaded into Kubernetes (提供实时的镜像升级，以及自定义 webhook，解决镜像升级的痛点。)
  + same chart can be installed several times into the same cluster; each will have it's own Release
+ Repository: a repository of published Charts (版本化、共享、Heml 私有仓库。)
+ Template: a Kubernetes configuration file mixed with Go/Sprig template

<!-- more -->

### Helm Components

+ Helm Client: Helm 是用户命令行工具，常与 CI/CD 部署在一起。它用来创建，拉取，搜索和验证 Charts，初始化 Tiller 服务。

+ Tiller: 是一个部署在Kubernetes集群内部的 Server，与 Helm client、Kubernetes APIServer 进行交互，管理这些应用的发布。

### Install Helm

**安装 Client:**

4Mac:

```zsh
➜  ~ brew install kubernetes-helm
```

4Linux:

[下载](https://github.com/kubernetes/helm) Helm

```bash
$ wget https://storage.googleapis.com/kubernetes-helm/helm-v2.8.2-linux-amd64.tar.gz
$ tar xzvf helm-v2.8.2-linux-amd64.tar.gz
$ mv linux-amd64/helm /usr/local/bin/

$ helm version
Client: &version.Version{SemVer:"v2.8.2", GitCommit:"a80231648a1473929271764b920a8e346f6de844", GitTreeState:"clean"}
Error: cannot connect to Tiller
```

**安装 Tiller:**

`$ helm init`

> 在安装 tiller 之前，需要在 10.0.77.16(ss,u know) 上配置好 kubectl 工具和 kubeconfig 文件，并确保可以正常访问 Kubernetes APIServer。

```bash
$ scp root@n1.k8s.com:/usr/local/bin/kubectl /usr/local/bin/
$ mkdir $HOME/.kube/config
$ scp root@n1.k8s.com:/root/.kube/config /root/.kube/

$ kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"} 
```

因为 k8s cluster(v1.9.2) 中开启了 `RBAC` 访问控制，所以需要创建 `service account: tiller` 并分配合适的角色。详情请参考 [TILLER AND ROLE-BASED ACCESS CONTROL](https://docs.helm.sh/using_helm/#tiller-and-role-based-access-control)。

```bash
$ kubectl create serviceaccount tiller --namespace kube-system
```

```json
$ cat helm-rbac.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
    
$ kubectl create -f helm-rbac.yaml
```

```bash
$ helm init --service-account tiller --skip-refresh
```

> GFW again，可以使用 `--tiller-image` 或`-i`参数来指定 tiller 要使用到的镜像。

```
gcr.io/kubernetes-helm/tiller:v2.8.2
--->
10.0.77.16/library/tiller:v2.8.2
```

```bash
$ helm init --service-account tiller --tiller-image 10.0.77.16/library/tiller:v2.8.2 --skip-refresh [--kube-context string]

Creating /root/.helm 
Creating /root/.helm/repository 
Creating /root/.helm/repository/cache 
Creating /root/.helm/repository/local 
Creating /root/.helm/plugins 
Creating /root/.helm/starters 
Creating /root/.helm/cache/archive 
Creating /root/.helm/repository/repositories.yaml 
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com 
Adding local repo with URL: http://127.0.0.1:8879/charts 
$HELM_HOME has been configured at /root/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
Happy Helming!
```

`--kube-context` - default '~/.kube/config'

```bash
$ kubectl get pod -n kube-system -l app=helm
NAME                            READY     STATUS    RESTARTS   AGE
tiller-deploy-b56587494-ctd9c   1/1       Running   0          1m

$ helm version
Client: &version.Version{SemVer:"v2.8.2", GitCommit:"a80231648a1473929271764b920a8e346f6de844", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.8.2", GitCommit:"a80231648a1473929271764b920a8e346f6de844", GitTreeState:"clean"}
```

```bash
更新仓库:

$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈ 

设置helm命令自动补全:

$ source <(helm completion zsh)
OR
$ source <(helm completion bash)
```

## Helm Chart Repository

> 当研发人员达到一定数量后，Docker 原生的镜像中心或 Harbor 成为瓶颈。[JFrog Artifactory](https://www.jfrog.com) 提供了企业内部的高可用 Docker 注册中心集群，能够提供高并发 Docker Pull 的拉取。
 
Artifactory 的虚拟 Helm Chart 仓库能够聚合公司本地和远程的仓库成为一个仓库，为开发者解析和安装 Charts 时提供唯一的 URL。如下图所示：




JFrog Helm Client 目前支持2个版本：

+ [匿名访问](https://github.com/kubernetes/helm) Artifactory
+ [认证访问](https://github.com/JFrogDev/helm) Artifactory

## Artifactory on K8s

[官方参考](https://github.com/JFrogDev/artifactory-docker-examples/tree/master/kubernetes)

Yaml 文件在[这里](https://github.com/acquaai/K8S/tree/master/yaml/JFrog-Artifactory)。

### Persistent Storage

```
$ cd artifactory/
$ kubectl create secret generic ceph-secret --type="kubernetes.io/rbd"   --from-literal=key='AQCwhKdaIbKRHxAAYfCmoj+VRsfQXSs0784Kow=='
$ kubectl create -f art-storageclass.yaml
```

### Database Driver

```bash
$ docker build -t 10.0.77.16/library/artifactory-pro-mysql:5.10.1 -f Dockerfile.mysql .
$ docker push 10.0.77.16/library/artifactory-pro-mysql:5.10.1
```

### MySQL Database

```bash
$ kubectl create -f mysql.yaml
```

### Artifactory Pro

```bash
$ kubectl create -f artifactory.yaml
$ kubectl logs -f artifactory-k8s-deployment-667d8847d-mrj2m
...
###########################################################
### Artifactory successfully started (93.102 seconds)   ###
###########################################################
```

### Nginx (option)

**SSL secret**

使用[michaelliao](https://github.com/michaelliao/itranswarp.js/blob/master/conf/ssl/gencert.sh)的脚本生成 demo SSL secret。

```bash
$ sh gen-ssl-cert.sh 
Enter your domain [www.example.com]: artifactory
...

$ kubectl create secret tls art-tls --cert=artifactory.crt --key=artifactory.key
```

```bash
$ kubectl create -f nginx.yaml
```

### Accessing Artifactory Pro

```bash
$ kubectl get pods
NAME                                         READY     STATUS    RESTARTS   AGE
artifactory-k8s-deployment-667d8847d-mrj2m   1/1       Running   0          7m
mysql-k8s-deployment-5746f95457-rpzc6        1/1       Running   0          10m
nginx-k8s-deployment-5cb5fdb4c4-fvn4s        1/1       Running   0          2m

$ kubectl get svc
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
artifactory         NodePort    10.254.85.186   <none>        8081:30809/TCP               8m
kubernetes          ClusterIP   10.254.0.1      <none>        443/TCP                      39d
mysql-k8s-service   ClusterIP   10.254.91.68    <none>        3306/TCP                     10m
nginx-k8s-service   NodePort    10.254.131.58   <none>        80:30021/TCP,443:31086/TCP   3m

$ kubectl describe svc/nginx-k8s-service
Name:                     nginx-k8s-service
Namespace:                default
Labels:                   app=nginx-k8s-service
                          group=artifactory-k8s
Annotations:              <none>
Selector:                 app=nginx-k8s-deployment
Type:                     NodePort
IP:                       10.254.131.58
Port:                     port-1  80/TCP
TargetPort:               80/TCP
NodePort:                 port-1  30021/TCP
Endpoints:                172.30.13.5:80
Port:                     port-2  443/TCP
TargetPort:               443/TCP
NodePort:                 port-2  31086/TCP
Endpoints:                172.30.13.5:443
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

### Config Artifactory Repository

[Master Your Helm Chart Repositories in Artifactory](https://jfrog.com/blog/master-helm-chart-repositories-artifactory/)

`admin` 用户登录`http://10.0.77.18:30809/`新建：

`Local Repository`，`Remote Repository`，然后新建`Virtual Repository`。注意，`Virtual Repository` 包含`本地和远程仓库`

![](/images/virtual.png)

![](/images/vir-local-remote.png)

### Helm Client Registry Repository

![](/images/resolving.png)

```bash
$ helm repo add helm-virtual ...
"helm-virtual" has been added to your repositories

$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "helm-virtual" chart repository
...Unable to get an update from the "stable" chart repository (https://kubernetes-charts.storage.googleapis.com):
        Get https://kubernetes-charts.storage.googleapis.com/index.yaml: dial tcp 172.217.160.112:443: i/o timeout
Update Complete. ⎈ Happy Helming!⎈ 

$ helm repo list
NAME            URL                                             
stable          https://kubernetes-charts.storage.googleapis.com
local           http://127.0.0.1:8879/charts                    
helm-virtual    http://10.0.77.18:30809/artifactory/helm-virtual
```
