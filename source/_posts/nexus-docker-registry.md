---
title: Nexus Docker Registry
date: 2019-08-06 11:41:11
categories: DevOps
---
Sonatype Nexus OSS 3 开始支持 Docker 镜像仓库，当拉取的 images 在 docker(hosted) 仓库中不存在时，会自动从配置的 docker(proxy) 仓库中拉取。


## Deploy Nexus

```zsh
$ sudo mkdir /nexus-data
$ sudo chmod -Rv 200 /nexus-data/
```

```zsh
$ docker run -dti \
        --net=host \
        --name=nexus \
        --privileged=true \
        --restart=always \
        -e INSTALL4J_ADD_VM_PARAMS="-Xms16g -Xmx16g -XX:MaxDirectMemorySize=24g" \
        -v /etc/localtime:/etc/localtime \
        -v /nexus-data:/nexus-data \
        sonatype/nexus3:3.18.1
```

<!-- more -->

```zsh
$ ll /nexus-data/admin.password
-rw-r--r-- 1 200 200 36 Sep  8 23:58 /nexus-data/admin.password
$ cat /nexus-data/admin.password
545cb562-a03f-4dc8-a78f-d9a15fa2442a
```

`http://ip:8081`


## Configure Blob Stores

+ aliyun-hub
+ default
+ docker-hub
+ google-hub
+ ybcard

## Configure Repositories

+ docker(group): hub-group 8082
 - docker(proxy): docker-hub 8083
 - docker(proxy): google-hub 8084
 - docker(proxy): aliyun-hub 8085
 - docker(hosted): ybcard 8086

### Create Repository: docker (proxy)

+ Name: docker-hub
* [x] Online:
+ HTTP: 8083
* [x] Enable Docker V1 API:
+ Remote storage: |
 + https://registry-1.docker.io
 + https://k8s.gcr.io
 + https://registry.cn-hangzhou.aliyuncs.com
+ Docker Index: Use Docker Hub
* [x] Auto blocking enabled:
+ Maximum component age: 1440
+ Maximum metadata age: 1440
+ Blob store: docker-hub
* [x] Strict Content Type Validattion:
* [x] Not found cache enabled:
+ Not found cache TTL: 1440


### Create Repository: docker (hosted)

+ Name: ybcard
* [x] Online:
+ HTTP: 8086
* [x] Enable Docker V1 API:
+ Blob store: ybcard
* [x] Strict Content Type Validattion:
+ Deployment policy: Allow redeploy


### Create Repository: docker (group)

+ Name: hub-group
* [x] Online:
+ HTTP: 8082
* [x] Enable Docker V1 API:
+ Blob store: default
* [x] Strict Content Type Validattion:
+ Members: docker-hub, aliyun-hub, google-hub, ybcard

`NOTE:` group 仓库并不能推送镜像，需要 **docker(hosted): ybcard 8086** 才能推送。


## Configure Nginx

为了 **pull** 不携带端口，使用[自签名泛域名证书](https://github.com/fishdrowned/ssl)。

Nexus WebUI: nexus.xxx.com
Docker registry: registry.xxx.com

**`nginx.conf`**:

```nginx
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
	worker_connections 768;
}

http {
	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	#keepalive_timeout 5 5;
   proxy_buffering off;

	include /etc/nginx/mime.types;
	default_type application/octet-stream;

        upstream nexus_web {
            server 172.31.16.10:8081;
        }

        upstream nexus_docker_pull {
            server 172.31.16.10:8082;
        }

        upstream nexus_docker_push {
            server 172.31.16.10:8086;
        }  

    server {
        listen 443 ssl;
 
        server_name registry.acqua.com;

        access_log /data/wwwlogs/registry.acqua.com.log;
        error_log /data/wwwlogs/registry.error.log;

        # allow large uploads of files - refer to nginx documentation
        client_max_body_size 1G;

        # optimize downloading files larger than 1G - refer to nginx doc before adjusting
        #proxy_max_temp_file_size 2048m;

        # required to avoid HTTP 411: see Issue #1486 (https://github.com/docker/docker/issues/1486)
        chunked_transfer_encoding on;

        ssl on;
        ssl_certificate     /etc/nginx/ssl/acqua.com.bundle.crt;
        ssl_certificate_key /etc/nginx/ssl/acqua.com.key.pem;


        # default push
        set $upstream "nexus_docker_push";

        # docker pull
        if ( $request_method ~* 'GET') {
             set $upstream "nexus_docker_pull";
        }

        # only docker(hosted)
        if ($request_uri ~ '/search') {
            set $upstream "nexus_docker_push";
        }

    location / {
            proxy_pass http://$upstream;
            proxy_set_header Host $host;
            proxy_send_timeout 3600;
            proxy_read_timeout 3600;
            proxy_connect_timeout 3600;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto "https";
        }
    }


    server {
        listen 443 ssl;
        server_name nexus.cs.acqua.com;

        access_log /data/wwwlogs/nexus.cs.acqua.com.log;
        error_log /data/wwwlogs/nexus.error.log;

        # allow large uploads of files - refer to nginx documentation
        client_max_body_size 1G;

        # optimize downloading files larger than 1G - refer to nginx doc before adjusting
        #proxy_max_temp_file_size 2048m;

        ssl on;
        ssl_certificate     /etc/nginx/ssl/acqua.com.bundle.crt;
        ssl_certificate_key /etc/nginx/ssl/acqua.com.key.pem;

        location /download {
            root /data/wwwroot;
        }

        location / {
            proxy_pass http://nexus_web;
            proxy_set_header Host $host;
            proxy_send_timeout 120;
            proxy_read_timeout 300;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto "https";
       }
    }


    gzip on;

	## Virtual Host Configs ##
	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;
}
```


## 导入证书

+ 将自签名的根证书 `out/root.crt`:
 - 导入客户端；
 - 导入所有需要访问镜像仓库的 `control plane` 和 `worker` 节点。

```zsh
$ sudo mkdir -p /etc/docker/certs.d/registry.xxx.com
$ cd /etc/docker/certs.d/registry.xxx.com
$ sudo curl -k https://nexus.cs.xxx.com/download/ca.crt -o ca.crt

k8s@cp-172-31-16-11:~$ docker login -u admin https://registry.xxx.com
Password:
WARNING! Your password will be stored unencrypted in /home/k8s/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

## **WARNING!**

Docker 利用 docker login 命令来校验用户镜像仓库的登录凭证，并不 Web Login，仅仅是一种登录试探校验 **用户名、密码** 是否正确，正确情况下 Docker 会把 *仓库域名、用户名、密码* 等信息进行 base64 编码后保存在 `$HOME/.docker/config.json` 文件中。

```zsh
$ cat /home/k8s/.docker/config.json
{
	"auths": {
		"registry.xxx.com": {
			"auth": "cGFzc3dvcmQ="
```

针对每一个镜像仓库，只保存最近一次有效登录的用户名和密码。docker logout 时从文件中删除相应信息。

用户密码解码：

```zsh
$ echo 'cGFzc3dvcmQ=' | base64 --decode
password
``` 

### 安全存储 docker login 登录凭证

Docker 官方为不同平台提供了相应的解决方案: 详见 [docker-credential-helpers](https://github.com/docker/docker-credential-helpers)
 
 
**Ref**

+ [Running The Nexus Platform Behind Nginx Using Docker](https://blog.sonatype.com/running-the-nexus-platform-behind-nginx-using-docker)
