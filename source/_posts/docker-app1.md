---
title: 构建Docker容器应用
date: 2018-01-25 09:29:15
categories: DevOps
---
## Redis容器应用

```bash
$ cat Dockerfile 

FROM ubuntu:14.04
MAINTAINER Acqua Young "acqua@gmail.com"
ENV REFRESHED_AT 2018-01-20

RUN apt-get update
RUN apt-get -y install redis-server redis-tools

EXPOSE 6379
ENTRYPOINT ["/usr/bin/redis-server"]
CMD []

$ docker build -t acqua/redis .
$ docker run -d -p 6379 --name redis acqua/redis

Docker默认会把公开的端口绑定到所有的网络接口上，localhost或127.0.0.1来访问。
```

<!-- more -->

## 容器互连

```bash
$ docker run -d --name redis acqua/redis
未指定端口

$ docker run -p 4567 \
  --name webapp --link redis:db -t -i \
  -v /root/sinatra/webapp:/opt/webapp acqua/sinatra \
  /bin/bash
```

> --link参数创建了两个容器间的父子连接。该参数需指定：一个是要连接的容器名字，另一个是连接后容器的别名。示例中，把新容器连接到redis容器，并使用db作为别名。别名可以访问公开信息，而无须关注底层容器的名字。连接让父容器能访问子容器，并且把子容器的一些连接细节分享给父容器，这些细节有助于配置应用程序并使用这个连接。

> 在启动Redis容器时，并没有使用-p公开Redis的端口，因为不需要。通过把容器连接在一起，可以让父容器直接访问任意子容器的公开端口（如，父容器webapp可以连接到子容器redis的6379端口）。更好的是只有使用--link参数连接到这个容器的容器才能连接到这个端口，容器的端口不需要对本地宿主机公开。

> 出于安全考虑，可以强制Docker只允许有连接的容器之间互相通信，需要在启动Docker守护进程时加上`--icc=false`标志，关闭所有没有连接的容器间的通信。

也可以把多个容器连接在一起。如想让这个Redis实例服务于多个Web应用程序，可以把每个Web应用程序的容器和同一个redis容器连接在一起：

```bash
$ docker run -p 4567 --name webapp2 --link redis:db ...
$ docker run -p 4567 --name webapp3 --link redis:db ...

被连接的容器必须运行在同一个Docker宿主机上，不同Docker宿主机上运行的容器无法连接。
```

Docker在父容器中2个地方写入了连接信息：

+ /etc/hosts
  
```bash
$ cat  /etc/hosts
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.17.0.4      db bb4ca80d2586 redis
172.17.0.3      004026a0d810

容器的主机名也可以不是其ID的一部分，可以在执行`docker run`命令时使用`-h`或`--hostname`参数指定容器的主机名。
```

+ 包含连接信息的环境变量中

```bash
root@004026a0d810:/# env
HOSTNAME=004026a0d810
DB_NAME=/webapp/db
DB_PORT_6379_TCP_PORT=6379
TERM=xterm
DB_PORT=tcp://172.17.0.4:6379
DB_PORT_6379_TCP=tcp://172.17.0.4:6379
OLDPWD=/opt
LS_COLORS=rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35:do=01;35:bd=40;33;01:cd=40;33;
......
DB_ENV_REFRESHED_AT=2018-01-20
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
REFRESHED_AT=2018-01-19
PWD=/
DB_PORT_6379_TCP_ADDR=172.17.0.4
DB_PORT_6379_TCP_PROTO=tcp
SHLVL=1
HOME=/root
LESSOPEN=| /usr/bin/lesspipe %s
LESSCLOSE=/usr/bin/lesspipe %s %s
_=/usr/bin/env
```

Docker在连接webapp和redis容器时，自动创建了以DB开头的环境变量，自动创建的环境变量包含以下信息：

+ 子容器的名字
+ 容器里运行的服务所使用的协议、IP和端口号
+ 容器里运行的不同服务所指定的协议、IP和端口号
+ 容器里由Docker设置的环境变量的值

### 使用容器连接来通信的方式

+ 使用环境变量  （发现服务的方法）
+ 使用DNS和/etc/hosts信息

## Jekyll & Apache应用

[Dockerfiles](https://github.com/acquaai/dockerjenkins/tree/master/Jekyll_Apache)

```bash
$ docker run -v /root/jekyll/acqua_blog:/data/ --name acqua_blog acqua/jekll

Configuration file: /data/_config.yml
            Source: /data
       Destination: /var/www/html
      Generating... 
                    done.
 Auto-regeneration: disabled. Use --watch to enable.

$ docker run -d -P --volumes-from acqua_blog acqua/apache
$ docker port 15a03f8210c8 80
0.0.0.0:32775

$ docker start acqua_blog
$ docker logs acqua_blog

因为共享卷会自动更新，不需要更新或重启Apache容器。
```

### 备份Jekyll卷

```bash
$ docker run --rm --volumes-from acqua_blog \
  -v $(pwd):/backup ubuntu \
  tar cvf /backup/acqua_blog.tar /var/www/html

--rm参数会在容器（Ubuntu）的进程运行完毕后自动删除，适用容器用完即扔的场景。
   
$ ls
acqua_blog  acqua_blog.tar  Dockerfile
```

## Java应用

[Dockerfiles](https://github.com/acquaai/dockerjenkins/tree/master/Java)

```bash
$ docker build -t acqua/java .
$ docker run  -t -i --name sample acqua/java \
 https://tomcat.apache.org/tomcat-7.0-doc/appdev/sample/sample.war

$ docker inspect -f '{{ .Config.Volumes }}' sample
map[/var/lib/tomcat7/webapps/:{}]

$ docker build -t acqua/tomcat7 .
$ docker run --name sample_app --volumes-from sample \
 -d -P acqua/tomcat7
 
$ docker port sample_app 8080
0.0.0.0:32768

http://x.x.x.x:32768/sample/
```

**Reference**
[The Docker Book](https://github.com/turnbullpress/dockerbook-code)
