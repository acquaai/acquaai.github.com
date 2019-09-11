---
title: Harbor (Updated)
date: 2019-08-06 15:05:45
categories: DevOps
---
Harbor是由VMware公司开源的基于Docker的企业级容器注册服务器，它包括权限管理(RBAC)、LDAP、日志审核、管理界面、自我注册、镜像复制和中文支持等功能。

## Install Docker

**Pass it over**.


## [Install Docker Compose](https://docs.docker.com/compose/install/#install-compose)

```zsh
$ sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
$ docker-compose --version
docker-compose version 1.24.1, build 4667896b
```


## Install Harbor with https

[Prerequisites for the Harbor](http://github.com/vmware/harbor)

```zsh
$ wget https://storage.googleapis.com/harbor-releases/release-1.8.0/harbor-offline-installer-v1.8.2.tgz
$ sudo tar xzvf harbor-offline-installer-v1.8.2.tgz  -C /
```

<!-- more -->

### Gen Harbor SSL

[自签泛域名证书](https://github.com/fishdrowned/ssl)


### Configure Harbor

**ssl**:

```zsh
$ ls -l /data/cert/
total 8
-rw-r--r-- 1 root root 2863 Sep 11 17:07 xxx.com.bundle.crt
-rw------- 1 root root 1675 Sep 11 17:08 xxx.com.key.pem
-rw-r--r-- 1 root root 1350 Sep 11 17:12 root.crt
```

**configuration**:

```yml
$ grep -Ev "^$|#" /harbor/harbor.yml
hostname: reg.xxx.com
http:
  port: 80
https:
  port: 443
  certificate: /data/cert/xxx.com.bundle.crt
  private_key: /data/cert/xxx.com.key.pem
harbor_admin_password: Harbor12345
database:
  password: root123
data_volume: /data
clair:
  updaters_interval: 12
  http_proxy:
  https_proxy:
  no_proxy: 127.0.0.1,localhost,core,registry
jobservice:
  max_job_workers: 10
chart:
  absolute_url: disabled
log:
  level: info
  rotate_count: 50
  rotate_size: 200M
  location: /var/log/harbor
_version: 1.8.0
uaa:
  ca_file: /data/cert/root.crt
```

**installing**:

```zsh
$ sudo -E env "PATH=$PATH" ./install.sh --with-notary --with-clair --with-chartmuseum
```

```log
...... installing log ......
✔ ----Harbor has been installed and started successfully.----

Now you should be able to visit the admin portal at https://x.x.x.x.
For more details, please visit https://github.com/goharbor/harbor .
```

**--with-notary**: 镜像签名
**--with-clair**: 漏扫
**--with-chartmuseum**: Helm chart


## Access Harbor

### Client Access

将自签名的 `root.crt` 证书拷贝到需要访问 Harbor 的 docker 主机的 /etc/docker/certs.d/`reg.xxx.com`/。

> If the Docker registry is accessed without a port number, do not add the port to the directory name. The following shows the configuration for a registry on default port 443 which is accessed with:

```zsh
$ ll /etc/docker/certs.d/reg.xxx.com/
root.crt

$ docker login -u admin reg.xxx.com
Password:
WARNING! Your password will be stored unencrypted in /home/k8s/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```


### Kubernetes Access

```bash
$ kubectl create secret docker-registry registrykey --docker-server=reg.xxx.com --docker-username=acqua --docker-password=Harbor12345 --docker-email=acqua@acqua.ai
secret "registrykey" created

$ kubectl get secret
NAME                  TYPE                                  DATA      AGE
registrykey           kubernetes.io/dockerconfigjson        1         33s

$ kubectl describe pods nginx-65486cc689-t6f7b
...
Events:
  Type    Reason                 Age   From                 Message
  ----    ------                 ----  ----                 -------
  ......
  Normal  Scheduled              2m    default-scheduler    10.0.77.17  pulling image "reg.xxx.com/acqua/nginx:1.9"
  Normal  Pulled                 2m    kubelet, 10.0.77.17  Successfully pulled image "reg.xxx.com/acqua/nginx:1.9"
  ......
```


## Maintain Harbor

使用 docker-compose [管理 Harbor](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md) 时必须在`docker-compose.yml`文件目录运行。

```bash
$ cd /harbor/
$ sudo docker-compose stop
$ sudo docker-compose start
```

Harbor 修改配置流程：

```bash
$ sudo docker-compose down -v
$ sudo vim harbor.yml
$ sudo ./prepare
$ sudo docker-compose up -d
```

删除 Harbor 的容器，但保留镜像和 Harbor 的数据库文件在文件系统上：

```bash
$ sudo docker-compose down -v
```

删除 Harbor 的数据库和镜像数据(重新安装的环境)：

```bash
$ sudo rm -r /data/database
$ sudo rm -r /data/registry
```

## Projects

**Project: acqua**

```bash
$ docker tag SOURCE_IMAGE[:TAG] reg.xxx.com/acqua/IMAGE[:TAG]
$ docker push reg.xxx.com/acqua/IMAGE[:TAG]
```


**Ref**

+ [configure_https](https://github.com/goharbor/harbor/blob/master/docs/configure_https.md)
+ [Generate self-signed certificates](https://coreos.com/os/docs/latest/generate-self-signed-certificates.html)
