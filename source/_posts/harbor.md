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

```zsh
$ grep -Ev "^$|#" /harbor/harbor.yml
```

```yml
# Required parameters
hostname: reg.xxx.com

data_volume: /data

harbor_admin_password: Harbor12345

database:
  password: root123

jobservice:
  max_job_workers: 10

log:
  level: info
  rotate_count: 50
  rotate_size: 200M
  location: /var/log/harbor

# optional parameters
http:
  port: 80
https:
  port: 443
  certificate: /data/cert/xxx.com.bundle.crt
  private_key: /data/cert/xxx.com.key.pem
clair:
  updaters_interval: 12
  http_proxy: http://10.30.1.99:1080
  https_proxy: http://10.30.1.99:1080
  no_proxy: 127.0.0.1,localhost,core,registry
chart:
  absolute_url: enabled
uaa:
  ca_file: /data/cert/root.crt
```

**installing**:

```zsh
$ sudo ./install.sh --with-notary --with-clair --with-chartmuseum
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


### [Kubernetes Access](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)


## Managing Harbor's lifecycle

Stopping/Starting Harbor:

```zsh
$ cd /harbor/
$ sudo docker-compose stop
$ sudo docker-compose start
```

To change Harbor's configuration:

```zsh
$ sudo docker-compose down -v
$ sudo vim harbor.yml
$ sudo ./prepare --with-notary --with-clair --with-chartmuseum
$ sudo docker-compose up -d
```

Removing Harbor's containers while keeping the image data and Harbor's database files on the file system:

```zsh
$ sudo docker-compose down -v
```

Removing Harbor's database and image data (for a clean re-installation):

```zsh
$ rm -r /data/database
$ rm -r /data/registry
```


## Projects

**Project: acqua**

```zsh
$ docker tag SOURCE_IMAGE[:TAG] reg.xxx.com/acqua/IMAGE[:TAG]
$ docker push reg.xxx.com/acqua/IMAGE[:TAG]
```


## How to process that forget Harbor's admin password

```zsh
$ docker exec -it harbor-db bash

root [ / ]# psql -h postgresql -d postgres -U postgres
Password for user postgres: 
psql (9.6.14)
Type "help" for help.

postgres=#
```

```sql
postgres=# \l
postgres=# \c registry
You are now connected to database "registry" as user "postgres".

registry=# \d

registry=# select user_id, username, password, realname, salt from harbor_user;
 user_id | username  |             password             |    realname    |               salt               
---------+-----------+----------------------------------+----------------+----------------------------------
       2 | anonymous |                                  | anonymous user | 
       3 | ybcard    | f5f92a94bfc48c36a539a644167476e0 | ybcard         | uc6dbjkwh178i1aycisi26q8y8mo593v
       1 | admin     | 4f16b74b68178d0c83d00af80ddb7d10 | system admin   | vsq9qbd0jgu3236iz0beat43yl9av11a

# pbkdf2 algorithm, "Admin123"

update harbor_user set password='e7c0331ebb021d64713c0515f6dad38f', salt='pa4mmop0v9lhnv2vpvmkuv941it72ku6' where username='admin';

registry=# \q
root [ / ]# exit
```


**Ref**

+ [configure_https](https://github.com/goharbor/harbor/blob/master/docs/configure_https.md)
+ [User Guide](https://github.com/goharbor/harbor/blob/master/docs/user_guide.md#user-guide)
+ [Generate self-signed certificates](https://coreos.com/os/docs/latest/generate-self-signed-certificates.html)
+ [pbkdf2](https://github.com/mitsuhiko/python-pbkdf2/blob/master/pbkdf2.py)
