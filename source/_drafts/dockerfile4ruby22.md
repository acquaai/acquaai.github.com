---
title: Dockerfile for ruby2.2
date: 2018-01-19 17:48:01
categories: DevOps
---
```bash
FROM ubuntu:14.04
MAINTAINER Acqua Young "xXx@gmail.com"
ENV REFRESHED_AT 2018-01-19
RUN apt-get update
RUN apt-get dist-upgrade -y

RUN DEBIAN_FRONTEND=noninteractive apt-get -y dist-upgrade
RUN DEBIAN_FRONTEND=noninteractive apt-get -y install python-software-properties
RUN DEBIAN_FRONTEND=noninteractive apt-get -y install software-properties-common
RUN DEBIAN_FRONTEND=noninteractive add-apt-repository ppa:brightbox/ruby-ng
RUN apt-get update
RUN DEBIAN_FRONTEND=noninteractive apt-get -y install ruby2.2 ruby2.2-dev build-essential redis-tools

RUN gem install --no-rdoc --no-ri sinatra json redis
RUN mkdir -p /opt/webapp
EXPOSE 4567
CMD [ "/opt/webapp/bin/webapp"]
```
<!-- more -->

```bash
$ docker build -t acqua/sinatra .
...
Setting up ruby2.2 (2.2.8-1bbox1~trusty1) ...
update-alternatives: using /usr/bin/gem2.2 to provide /usr/bin/gem (gem) in auto mode
update-alternatives: using /usr/bin/ruby2.2 to provide /usr/bin/ruby (ruby) in auto mode
Setting up ruby2.2-dev:amd64 (2.2.8-1bbox1~trusty1) ...
Processing triggers for libc-bin (2.19-0ubuntu6.14) ...
 ---> b35b1038951b
Removing intermediate container fb5cfe9c853c
Step 12 : RUN gem install --no-rdoc --no-ri sinatra json redis
 ---> Running in 36f66c2eb971
Successfully installed mustermann-1.0.1
Successfully installed rack-2.0.3
Successfully installed rack-protection-2.0.0
Successfully installed tilt-2.0.8
Successfully installed sinatra-2.0.0
Building native extensions.  This could take a while...
Successfully installed json-2.1.0
Successfully installed redis-4.0.1
7 gems installed
 ---> 6995f2e94b08
Removing intermediate container 36f66c2eb971
Step 13 : RUN mkdir -p /opt/webapp
 ---> Running in 3971b011bba9
 ---> 0d52e86f3cf3
Removing intermediate container 3971b011bba9
Step 14 : EXPOSE 4567
 ---> Running in 63c9b30c9030
 ---> aef6d228461d
Removing intermediate container 63c9b30c9030
Step 15 : CMD /opt/webapp/bin/webapp
 ---> Running in c98d207ffba6
 ---> eace9e0a84fd
Removing intermediate container c98d207ffba6
Successfully built eace9e0a84fd
```
