---
title: Harbor
date: 2018-02-09 11:05:45
categories: DevOps
---
Harbor是由VMware公司开源的基于Docker的企业级容器注册服务器，它包括权限管理(RBAC)、LDAP、日志审核、管理界面、自我注册、镜像复制和中文支持等功能。

## [安装Docker](https://docs.docker.com/install/)

Docker从`v17.03`开始分为 CE（[Community Edition](https://docs.docker.com/release-notes/docker-ce/)）和 EE（Enterprise Edition）。

**1. 配置Repository**

```bash
$ sudo yum install -y yum-utils device-mapper-persistent-data lvm2
$ sudo yum-config-manager
   --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
<!-- more -->

**Optional**

```bash
$ sudo yum-config-manager --enable docker-ce-edge/docker-ce-test
```

**2. Install Docker CE**

```bash
$ yum list docker-ce --showduplicates | sort -r
$ sudo yum install docker-ce 最新版本
$ sudo yum install docker-ce-17.06.1.ce 指定版本
```

`Docker is installed but not started. The docker group is created, but no users are added to the group.`

**3. Add User to Docker Group**

docker组内用户执行命令的时候会自动在所有命令前添加`sudo`.

```bash
$ sudo usermod -aG docker $(whoami)
```

Re-login:

```
$ sudo systemctl start docker
$ docker run hello-world
$ sudo systemctl enable docker
```

## 安装Docker Compose

[Install Docker Compose](https://docs.docker.com/compose/install/#install-compose)

```bash
$ sudo curl -L https://github.com/docker/compose/releases/download/1.19.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose

$ docker-compose --version
docker-compose version 1.19.0, build 9e633ef
```

## 安装Harbor启用https

[Prerequisites for the Harbor](http://github.com/vmware/harbor)

```bash
$ wget https://storage.googleapis.com/harbor-releases/release-1.4.0/harbor-offline-installer-v1.4.0.tgz
$ sudo tar xvf harbor-offline-installer-v1.4.0.tgz  -C /
```

### 安装CFSSL

```bash
$ sudo yum install -y git gcc
$ wget https://dl.google.com/go/go1.9.4.linux-amd64.tar.gz
$ sudo tar xzvf go1.9.4.linux-amd64.tar.gz -C /usr/local/

$ sudo vi /etc/profile
export GOPATH=/usr/local
export PATH=$PATH:/usr/local/go/bin
```

```bash
$ sudo go get -u github.com/cloudflare/cfssl/cmd/...
$ sudo ls /usr/local/bin
cfssl  cfssl-bundle  cfssl-certinfo  cfssljson  cfssl-newkey  cfssl-scan  mkbundle  multirootca
```

### 创建Harbor证书

**创建ca-config.json文件**

```json
$ sudo mkdir /root/cfssl && cd /root/cfssl
$ sudo vi ca-config.json
{
    "signing": {
        "default": {
            "expiry": "8760h"
        }, 
        "profiles": {
            "kubernetes": {
                "usages": [
                    "signing", 
                    "key encipherment", 
                    "server auth", 
                    "client auth"
                ], 
                "expiry": "8760h"
            }
        }
    }
}
```

**创建文件ca-csr.json文件**

```json
$ sudo vi ca-csr.json
{
    "CN": "kubernetes", 
    "key": {
        "algo": "rsa", 
        "size": 2048
    }, 
    "names": [
        {
            "C": "CN", 
            "ST": "Shenzhen", 
            "L": "Shenzhen", 
            "O": "k8s", 
            "OU": "System"
        }
    ]
}
```

**生成CA证书和私钥**

```bash
$ sudo cfssl gencert -initca ca-csr.json | cfssljson -bare ca
$ sudo ls ca*
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
```

**创建harbor-csr.json**

```json
$ sudo vi harbor-csr.json
{
    "CN": "kubernetes", 
    "hosts": [
        "127.0.0.1", 
        "10.0.77.16"
    ], 
    "key": {
        "algo": "rsa", 
        "size": 2048
    }, 
    "names": [
        {
            "L": "Shenzhen", 
            "O": "k8s", 
            "OU": "System"
        }
    ]
}
```

```bash
$ sudo cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes harbor-csr.json | cfssljson -bare harbor
$ sudo ls harbor*
harbor.csr  harbor-csr.json  harbor-key.pem  harbor.pem

$ mkdir -p /data/cert
$ sudo mv harbor*.pem /data/cert/
```

### 修改harbor.cfg文件

```bash
$ cd /harbor/
$ sudo vi harbor.cfg

hostname = 
db_password = 
harbor_admin_password = 
auth_mode = db_auth = 
self_registration =
project_creation_restriction =

**https**
ui_url_protocol = https
ssl_cert = /data/cert/harbor.pem
ssl_cert_key = /data/cert/harbor-key.pem
```

```bash
$ sudo -E env "PATH=$PATH" ./install.sh --with-notary
...
✔ ----Harbor has been installed and started successfully.----

Now you should be able to visit the admin portal at https://10.0.77.16. 
For more details, please visit https://github.com/vmware/harbor.
```

将自签名的 CA 证书拷贝到需要访问 Harbor 仓库的 docker 主机的 /etc/docker/certs.d/`registry-hostname`/下。

> If the Docker registry is accessed without a port number, do not add the port to the directory name. The following shows the configuration for a registry on default port 443 which is accessed with:

```bash
shell> mkdir -p /etc/docker/certs.d/10.0.77.16
shell> cp /etc/kubernetes/ssl/ca.pem /etc/docker/certs.d/10.0.77.16/ca.crt
```

**Client Access 1/3 Ways**

```bash
shell> kubectl create secret docker-registry registrykey --docker-server=10.0.77.16 --docker-username=acqua --docker-password=Harbor12345 --docker-email=acqua@acqua.ai
secret "registrykey" created

shell> kubectl get secret
NAME                  TYPE                                  DATA      AGE
registrykey           kubernetes.io/dockerconfigjson        1         33s

shell> kubectl describe pods nginx-65486cc689-t6f7b
...
Events:
  Type    Reason                 Age   From                 Message
  ----    ------                 ----  ----                 -------
  Normal  Scheduled              2m    default-scheduler    Successfully assigned nginx-65486cc689-t6f7b to 10.0.77.17
  Normal  SuccessfulMountVolume  2m    kubelet, 10.0.77.17  MountVolume.SetUp succeeded for volume "default-token-ctbfl"
  Normal  Pulling                2m    kubelet, 10.0.77.17  pulling image "10.0.77.16/acquaai/nginx:1.9"
  Normal  Pulled                 2m    kubelet, 10.0.77.17  Successfully pulled image "10.0.77.16/acquaai/nginx:1.9"
  Normal  Created                2m    kubelet, 10.0.77.17  Created container
  Normal  Started                2m    kubelet, 10.0.77.17  Started container
```

**Client Access 2/3 Ways**

如果10.0.88.16主机访问Harbor仓库，需要将Harbor的证书拷贝过来：

```bash
$ mkdir -p /etc/docker/certs.d/10.0.77.16 && cd /etc/docker/certs.d/10.0.77.16
$ scp -P 8086 root@10.0.77.16:/etc/docker/certs.d/10.0.77.16/ca.crt ./
$ docker login 10.0.77.16 
Username: acqua
Password: 
Login Succeeded
```

**参考**

+ [configure_https](https://github.com/vmware/harbor/blob/master/docs/configure_https.md)
+ [配置Harbor启用https和外部数据库](https://blog.frognew.com/2017/06/config-harbor-with-https-and-external-db.html)
+ [Kubernetes从Private Registry中拉取容器镜像的方法](https://tonybai.com/2016/11/16/how-to-pull-images-from-private-registry-on-kubernetes-cluster/)

## 管理Harbor

使用docker-compose[管理Harbor](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md)时必须在`docker-compose.yml`文件目录运行。

```bash
$ cd /harbor/
$ docker-compose stop
$ docker-compose start
```

Harbor修改配置流程：

```bash
$ docker-compose down -v
$ sudo vim harbor.cfg
$ sudo ./prepare
$ docker-compose up -d
```

删除Harbor的容器，但保留镜像和Harbor的数据库文件在文件系统上：

```bash
$ docker-compose down -v
```

删除Harbor的数据库和镜像数据(重新安装的环境)：

```bash
$ sudo rm -r /data/database
$ sudo rm -r /data/registry
```

### 创建私有项目

新建私有项目`acquaai`

Push Image到该项目的命令：

```bash
shell> docker tag SOURCE_IMAGE[:TAG] 10.0.77.16/acquaai/IMAGE[:TAG]
shell> docker push 10.0.77.16/acquaai/IMAGE[:TAG]
```


