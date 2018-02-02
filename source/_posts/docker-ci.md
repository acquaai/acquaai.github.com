---
title: Docker CI
date: 2018-01-25 11:20:43
categories: DevOps
---
![](/images/j_CI.png)
图片来源Google

<!-- more -->

## 构建Jenkins和Docker服务器

### Create Image

```bash
$ cat Dockerfile

FROM ubuntu:14.04
MAINTAINER Acqua Young "acqua@gmail.com"
ENV REFRESHED_AT 2018-01-20

RUN apt-get -y update
RUN apt-get install -qqy python-software-properties software-properties-common curl libcurl3 libcurl3-dev apt-transport-https ca-certificates
RUN curl -fsSL https://get.docker.io/gpg | apt-key add -
RUN echo deb https://get.docker.io/ubuntu docker main > /etc/apt/sources.list.d/docker.list
RUN add-apt-repository ppa:openjdk-r/ppa
RUN apt-get update -y && apt-get install -qqy iptables openjdk-8-jdk lxc lxc-docker git-core

ENV JENKINS_HOME /opt/jenkins/data
ENV JENKINS_MIRROR http://mirrors.jenkins-ci.org

RUN mkdir -p $JENKINS_HOME/plugins
RUN curl -sf -o /opt/jenkins/jenkins.war -L $JENKINS_MIRROR/war-stable/latest/jenkins.war

RUN for plugin in chucknorris greenballs scm-api git-client git ws-cleanup; \
    do curl -sf -o $JENKINS_HOME/plugins/${plugin}.hpi \
      -L $JENKINS_MIRROR/plugins/${plugin}/latest/${plugin}.hpi ; done

ADD ./dockerjenkins.sh /usr/local/bin/dockerjenkins.sh
RUN chmod +x /usr/local/bin/dockerjenkins.sh

VOLUME /var/lib/docker

EXPOSE 8080
ENTRYPOINT ["/usr/local/bin/dockerjenkins.sh"]
```

```bash
$ ls jenkins/
Dockerfile  dockerjenkins.sh

$ docker build -t acqua/dockerjenkins .
```

### Run Container

```bash
$ docker run -d -p 8080:8080 --name jenkins --privileged acqua/dockerjenkins
--privileged参数允许以其宿主机具有的（几乎）所有能力来运行容器，包括一些内核特性和设备访问。

$ docker logs jenkins
......
INFO: 
*************************************************************
*************************************************************
*************************************************************
Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:
865688c1608e470aa96de47b481b34df
This may also be found at: /opt/jenkins/data/secrets/initialAdminPassword
*************************************************************
*************************************************************
*************************************************************
INFO: Jenkins is fully up and running
......
```

## Jenkins Job

http://x.x.x.x:8080   **/初始密码日志中已列出。

> Jenkins Plugins:
> https://updates.jenkins-ci.org/download/plugins/
> http://mirrors.jenkins-ci.org/plugins/

### New Job

![](/images/j_Job.png)

### General

![](/images/j_WS.png)

### [Source Code Management](https://github.com/acquaai/dockerjenkins.git)

![](/images/j_Git.png)

### Build

Execute shell
Command

```bash
# Build the image to be used for this job.
IMAGE=$(docker build . | tail -1 | awk '{ print $NF }')

# Build the directory to be mounted into Docker.
MNT="$WORKSPACE/.."

# Execute the build inside Docker.
CONTAINER=$(docker run -d -v $MNT:/opt/project/ $IMAGE /bin/bash -c 'cd /opt/project/workspace && rake spec')

# Attach to the container so that we can see the output.
docker attach $CONTAINER

# Get its exit code as soon as the container stops.
RC=$(docker wait $CONTAINER)

# Delete the container we've just used.
docker rm $CONTAINER

# Exit with the same value as that with which the process exited.
exit $RC
```

### Post-build Actions

![](/images/j_Report.png)

### Run Job

![](/images/j_Build.png)

**Build History**下会列出构建的历史信息，`绿色Ball`表示成功，红色，sad。
每一条构建历史信息中的`Console Output`会显示构建的详细过程。

![](/images/j_C_output.png)

**Reference**
[The Docker Book](https://github.com/turnbullpress/dockerbook-code)
