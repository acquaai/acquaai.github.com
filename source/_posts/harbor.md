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
```

## 安装Docker Compose

[Install Docker Compose](https://docs.docker.com/compose/install/#install-compose)

```bash
$ sudo curl -L https://github.com/docker/compose/releases/download/1.19.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose

$ docker-compose --version
docker-compose version 1.19.0, build 9e633ef
```

## 安装Harbor

[Prerequisites for the Harbor](http://github.com/vmware/harbor)

```bash
$ wget https://storage.googleapis.com/harbor-releases/release-1.4.0/harbor-offline-installer-v1.4.0.tgz
$ sudo tar xvf harbor-offline-installer-v1.4.0.tgz  -C /
$ cd /harbor/
$ sudo vi harbor.conf

hostname = 
db_password = 
harbor_admin_password = 
auth_mode = db_auth = 
self_registration =
project_creation_restriction = 
```

```bash
$ sudo ./install.sh

May be:
$ sudo -E env "PATH=$PATH" ./install.sh

$ docker-compose ps
       Name                     Command               State                                Ports                              
------------------------------------------------------------------------------------------------------------------------------
harbor-adminserver   /harbor/start.sh                 Up                                                                      
harbor-db            /usr/local/bin/docker-entr ...   Up      3306/tcp                                                        
harbor-jobservice    /harbor/start.sh                 Up                                                                      
harbor-log           /bin/sh -c /usr/local/bin/ ...   Up      127.0.0.1:1514->10514/tcp                                       
harbor-ui            /harbor/start.sh                 Up                                                                      
nginx                nginx -g daemon off;             Up      0.0.0.0:443->443/tcp, 0.0.0.0:4443->4443/tcp, 0.0.0.0:80->80/tcp
registry             /entrypoint.sh serve /etc/ ...   Up      5000/tcp                                                        
```

使用admin/`harbor_admin_password =`登录：http://10.0.77.16/harbor/sign-in

[Configure https access](https://github.com/vmware/harbor/blob/master/docs/configure_https.md)

### Managing Harbor's lifecycle

使用docker-compose[管理Harbor]((https://github.com/vmware/harbor/blob/master/docs/installation_guide.md))时必须在`docker-compose.yml`文件目录运行。

```bash
$ docker-compose stop
$ docker-compose start
```

Harbor修改配置流程：

```bash
$ docker-compose down -v
$ sudo vim harbor.cfg
$ sudo prepare
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
