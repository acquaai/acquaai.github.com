---
title: Docker基础学习
date: 2018-01-24 18:53:41
categories: DevOps
---
## Install

[Docker CE](https://docs.docker.com/engine/installation/linux/docker-ce/centos/)

## Commonly Commands

`$ docker run -ti --name  ubuntu /bin/bash`
基于ubuntu镜像启动一个新的new_C容器，并在容器中启动Bash Shell。

<!-- more -->

`$ docker exec -d new_C touch /etc/newfile`
-d 表明需要运行一个后台进程，-d 参数之后，指定要在内部执行这个命令的容器名字以及要执行的命令。

`$ docker exec -ti new_C /bin/bash`
-ti 参数为执行的进程创建TTY并捕捉STDIN。

`$ docker ps -n 2`
显示最后2个容器，无论这些容器运行还是停止。

```bash
$ docker run --restart=always --name new_C -d ubuntu /bin/sh -c "while true; \
    do echo Hello World; sleep 1; done"
容器自动重；--restart=on-failure:3 尝试重启3次。
```

`$ docker inspect new_C`
查看容器信息。

```bash
$ docker inspect --format='{{ .NetworkSettings.IPAddress }}' new_C
172.17.0.2
--format 查看指定信息
```

`$ docker rm new_C`
删除容器，-f 强制删除活动容器。

`$ docker rm $(docker ps -aq)`
删除所有非活动容器。

`$ docker rmi acqua/test_web`
`$ docker rmi $(docker images -aq)`
删除镜像。

## Docker镜像和仓库

### Docker镜像

> Docker镜像是由文件系统叠加而成。最底端是一个引导文件系统，即bootfs。Docker用户几乎不会与此有交互。当一个容器启动后，它将会被移到内存中，而引导文件系统则会umount，以留出更多的内存供initrd磁盘镜像使用。在Docker里，rootfs永远只能是只读状态，并且Docker利用联合加载（union mount）技术又会在rootfs层上加载更多的只读文件系统。联合加载指的是一次同时加载多个文件系统，但是外表看起来只能看到一个文件系统。联合加载会将各层文件系统叠加到一起，这样最终的文件系统会包含所有底层的文件和目录。Docker将这样的文件系统称为镜像。当从一个镜像启动容器时，Docker会在该镜像的最顶层加载一个读写文件系统，程序就是在这个读写层中执行。

`Copy on write`
Docker第一次启动一个容器时，初始的读写层是空的。当文件系统发生变化时，这些变化都会应用到这一层上。比如，修改一个文件，这个文件首先从该读写层下面的只读层复制到该读写层。该文件的只读版本依然存在，但是已经被读写层中的该文件副本所隐藏。

`image-layering framework`

+ 显示Docker镜像

```bash
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
docker.io/ubuntu    latest              00fd29ccc6f1        4 weeks ago         110.5 MB
docker.io/centos    latest              3fa822599e10        6 weeks ago         203.5 MB

$ docker images centos
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
docker.io/centos    latest              3fa822599e10        6 weeks ago         203.5 MB
```

+ 拉取ubuntu仓库中所有内容

```bash
$ docker pull ubuntu
$ docker pull ubuntu:16.04

$ docker search mysql
INDEX       NAME                             DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
docker.io   docker.io/mysql                  MySQL is a widely used, open-source relati...   5545      [OK]       
docker.io   docker.io/mariadb                MariaDB is a community-developed fork of M...   1718      [OK]       
docker.io   docker.io/mysql/mysql-server     Optimized MySQL Server Docker images. Crea...   381                  [OK]

$ docker pull docker.io/mysql
```

+ commit 创建镜像
 
```bash
$ docker commit eb4a97d5a48c acqua/apache2
$ docker commit -m="custom image" --author="acqua" eb4a97d5a48c acqua/apache2:webserver
```

+ Dockerfile 创建镜像

```bash

$ mkdir test_web && cd test_web
$ vi Dockerfile

# Version: 0.0.1
FROM ubuntu:14.04    **/FROM 指定基础镜像（base image），后续指令都基于该镜像进行。
MAINTRAINER Acqua Young "acqua@gmail.com"
RUN apt-get update
RUN apt-get install -y nginx    **/ RUN ["apt-get","install","-y","nginx"]
RUN echo 'Hi, I am in your container' > /usr/share/nginx/html/index.html
EXPOSE 80

$ docker build -t="acqua/test_web" .
```

```bash

$ docker build -t="acqua/test_web:v1" .
如果没有制定任何标签，Docker自动为镜像设置一个latest标签。. 当前目录查找Dockerfile，
也可以用Git仓库源地址git@github.com:acqua/docker_test_web

$ docker history new_C
查看容器创建历程。
```

### 从镜像启动容器

```bash
$ docker run -d -p 80 --name test_web acqua/test_web nginx -g "daemon off;"
Docker可以在宿主机上随机选择一个介于 49000~49900 中一个比较大的端口号来映射到容器中的80端口上。

$ docker run -d -p 127.0.0.1:80:80 --name test_web acqua/test_web nginx -g "daemon off;"
也可以在Docker宿主机上指定个一个具体的端口号来映射到容器中的80端口上。

$ docker run -d -p 127.0.0.1::80 --name test_web acqua/test_web nginx -g "daemon off;"
容器内的80端口绑定到宿主机的随机端口上。

$ docker port 430a5ba5df0e 80
0.0.0.0:32768
查看容器内的 80 端口对应宿主机的端口。

$ docker run -d -P --name test_web acqua/test_web nginx -g "daemon off;"
-P（大写）会将容器内的80端口对本地宿主机公开，并且绑定到宿主机的一个随机端口上。
命令会将用来构建该镜像的Dockerfile文件中EXPOSE指令指定的其它端口也一并公开。

有了这个端口号，就可以使用本地宿主机的IP或127.0.0.1的localhost连接到运行中的容器，查看WEB服务器内容了。
$ docker port 0730f9d8b19f
5000/tcp -> 0.0.0.0:32769

$ curl localhost:32769
```

### Dockerfile 指令

+ CMD
  CMD用于指定容器启动时要运行的命令，RUN指令是指定镜像被构建时要运行的命令，而CMD是指定容器被启动时要运行的命令。
  `CMD ["/bin/bash", "-l"]`
  注：docker run命令可以覆盖CMD指令。
  在Dockerfile中只能指定一条CMD指令。如果指定了多条CMD指令，也只有最后一条CMD指令会被使用。
  
+ ENTRYPOINT
  与CMD指令非常相似，而ENTRYPOINT指令提供的命令则不容易在启动容器时被覆盖。实际上，docker run命令行中指定的任何参数都会被当做参数再次传递给ENTRYPOINT指令中指定的命令。
  `ENTRYPOINT ["/usr/sbin/nginx", "-g", "daemon off;"]`
  通过以数组的方式指定 ENTRYPOINT 在想运行的命令前加入 /bin/sh -c 避免各种问题。
  `ENTRYPOINT ["/usr/sbin/nginx"]`
  `CMD ["-h"]`
  此时当我们启动一个容器时，任何在命令行中指定的参数都会被传递给Ngnix守护进程。

  `docker run --entrypoint`来覆盖ENTRYPOINT指令。

+ WORKDIR
  用来在从镜像创建一个新容器时，在容器内部设置一个工作目录，ENTRYPOINT和 / 或CMD指定的程序会在这个目录下执行。
  `WORKDIR /opt/webapp/db`
  `RUN bundle install`
  `WORKDIR /opt/webapp`
  `ENTRYPOINT [" rackup"]`
  
  `docker run -w /var/log ubuntu pwd`来覆盖WORKDIR指令。
  
+ ENV
  用来在镜像构建过程中设置环境变量。
  `ENV RVM_PATH /home/rvm/`
  
  新的环境变量可以在后续任何RUN指令中使用，如同在命令前面指定了环境变量前缀一样：
  `RUN gem install unicorn`
  `ENV TARGET_DIR /opt/app`
  `WORKDIR $TARGET_DIR`
  `docker run -e`来传递环境变量
  
  `$ docker run -it -e "WEB_PORT=8080" ubuntu env`
  
+ USER
  指定该镜像以什么样的的用户去运行
  `USER nginx`
  `USER user:group`
  `USER uid:gid`
  
  `docker run -u`来覆盖指令，如果不指定默认为root。
  
+ VOLUME
  用来向基于镜像创建的容器添加卷。
  + 卷可以在容器间共享和重用
  + 一个容器可以不是必须和其它容器共享卷
  + 对卷的修改是立即生效
  + 对卷的修改不会对更新镜像产生影响
  + 卷会一直存在直到没有任何容器再使用它
  
  `VOLUME ["/opt/project", "/data"]`
  
+ ADD
  用来将构建环境下的文件和目录复制到镜像中。
  ADD software.lic /opt/application/software.lic
  将构建目录下的software.lic文件复制到镜像中的/opt/application/software.lic。源文件的位置参数可以是一个URL，或者构建上下文或环境中文件名或者目录。不能对构建目录或者上下文之外的文件进行ADD操作。
  `ADD http://wordpress.org/latest.zip /root/wordpress.zip`
  
  如果目的位置不存在，Docker会创建这个全路径（类似mkidr -p）。新创建的文件和目录的模式为0755，UID和GID为0。
  
+ COPY
  类似ADD，不同是COPY只关心在构建上下文中复制本地文件，而不会做文件提取（extraction）解压（decompression）
  `COPY conf.d /etc/apache2/`
  
  该指令创建的文件或目录的UID和GID都会设置为0，创建目录与mkdir -p相同。
  
+ ONBUILD
  为镜像添加触发器（trigger）。触发器会在构建过程中插入新指令，可以认为这些指令是紧跟在FROM之后指定的，触发器可以是任何构建指令：
  `ONBUILD ADD . /app/src`
  `ONBUILD RUN cd /app/src && make`
  
  Example:
  `FROM ubuntu:14.04`
  `MAINTAINER Acqua Young "xxx@mail.com"`
  `RUN apt-get update`
  `RUN apt-get install -y apache2`
  `ENV APACHE_RUN_USER www-data`
  `ENV APACHE_RUN_GROUP www-data`
  `ENV APACHE_LOG_DIR /var/log/apache2`
  `ONBUILD ADD . /var/www/`
  `EXPOSE 80`
  `ENTRYPOINT ["/usr/sbin/apache2"]`
  `CMD ["-D", "FOREGROUND"]`
  
  不能用在ONBUILD指令中，FROM、MAINTAINER和ONBUILD本身，也防止递归调用。
  
**Reference**
[The Docker Book](https://github.com/turnbullpress/dockerbook-code)
