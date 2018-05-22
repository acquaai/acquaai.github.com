---
title: 部署 Tornado 开发环境
date: 2018-05-22 14:54:48
categories: Python
---
`Tornado`：[Tornado](http://www.tornadoweb.org/en/stable/) 是 Python 编写出来的一个极轻量级、高可伸缩性和非阻塞IO的Web服务器软件，例如前 Friendfeed 网站。

`Supervisor`：一个服务（进程）管理工具，主要用于监控服务器上的服务，并且在出现问题时自动重启。

`Nginx`：在这里作为反向代理。

<!-- more -->

## CentOS 系统基础环境

```bash
$ yum update -y
```

因为目前 Supervisor 仅支持 python2，但又需要使用 Python3 来开发，所以它们需要共存。为了不同环境互不干扰，将使用`virtualenv`来达到这一目的。

### 安装 Python 3

```bash
# Python2的软链接

$ ll /usr/bin/python*
lrwxrwxrwx 1 root root    7 May 21 10:30 /usr/bin/python -> python2
lrwxrwxrwx 1 root root    9 May 21 10:30 /usr/bin/python2 -> python2.7
-rwxr-xr-x 1 root root 7216 Apr 11 15:36 /usr/bin/python2.7

# 安装Python3的依赖
$ yum install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gcc make 

# 安装Python3.6
$ wget https://www.python.org/ftp/python/3.6.5/Python-3.6.5.tgz
$ tar xzvf Python-3.6.5.tgz
$ cd Python-3.6.5
$ ./configure perfix=/usr/local/python3
$ make && make install
$ python -V

# （可选）设置Python3为系统默认版本

$ mv /usr/bin/python /usr/bin/python.bak
$ ln -s /usr/local/python3/bin/python3.6 /usr/bin/python
$ python -V

# 配置 yum（yum包管理工具使用Python2）
$ vi /usr/bin/yum

#! /usr/bin/python ===> #! /usr/bin/python2

$ vi /usr/libexec/urlgrabber-ext-down
#! /usr/bin/python ===> #! /usr/bin/python2
```

## 安装 Tornado

```bash
$ pip3 install tornado

# 新建示例页面
$ mkdir /var/www
$ vi /var/www/index.py
```

```py
import tornado.ioloop
import tornado.web
 
class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("Hello, Tornado!")
 
application = tornado.web.Application([
    (r"/", MainHandler),
    (r"/index.py", MainHandler),
])
 
if __name__ == "__main__":
    application.listen(8006)
    tornado.ioloop.IOLoop.instance().start()
```

### Test

```bash
$ python3 index.py

http://10.0.77.119:8006
```

## 安装 Nginx

```bash
$ cat /etc/yum.repos.d/nginx.repo
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
enabled=1

$ yum install nginx
```

[nginx signing key](http://nginx.org/keys/nginx_signing.key)

```bash
$ cat /etc/nginx/nginx.conf          

user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
    use epoll;
}

http {
    upstream tornado {
        server 10.0.77.119:8006;
    }

    server {
        listen 80;
        root /var/www;
        index index.py index.html;

        server_name server;

        location / {
            if (!-e $request_filename) {
                rewrite ^/(.*)$ /index.py/$1 last;
            }
        }

        location ~ /index\.py {
            proxy_pass_header Server;
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Scheme $scheme;
            proxy_pass http://tornado;
        }
    }
}
```

![](/images/test_Tornado.png)

## 安装 Supervisor

### pip2

+ 安装 pip
 + yum 安装
 
 ```bash
 
 $ yum install epel-release
 $ yum install python-pip
 ```
 
 + curl 安装

 ```bash
 $ curl "https://bootstrap.pypa.io/get-pip.py" -o "get-pip.py"
 $ python get-pip.py 
 ```

+ 检测安装

```bash
pip --help
pip -V
```

### virtualenv

```bash
$ pip2 install virtualenv
```

### 配置 supervisor

```bash
# 新建 virtualenv 环境
$ mkdir /etc/supervisor
$ virtualenv --distribute -p /usr/bin/python2 supervisor

Already using interpreter /usr/bin/python2
New python executable in /etc/supervisor/bin/python2
Also creating executable in /etc/supervisor/bin/python
Installing setuptools, pip, wheel...done.

# 进入 virtualenv 环境
$ cd supervisor/
$ source bin/activate
(supervisor) [root@tornado-web supervisor]# ./bin/pip install supervisor
...
Successfully installed meld3-1.0.2 supervisor-3.3.4


(supervisor) [root@tornado-web supervisor]# echo_supervisord_conf > /etc/supervisord.conf

# 修改如下内容：
[unix_http_server]
file=/var/run/supervisor.sock
chmod=0700

# 开启Web管理功能
[inet_http_server]
port=10.0.77.119:9001iface
username=user
password=123

[supervisord]
logfile=/var/log/supervisor/supervisord.log
pidfile=/var/run/supervisord.pid

[supervisorctl]
serverurl=unix:///var/run/supervisor.sock

# 在文件末尾增加如下内容：
# 主要作用就是在Supervisor启动的时候自动启动 hello 应用对应的
# Tornado Web Server进程并纳入管理。
[program:hello]
command=/usr/local/bin/python3.6 /var/www/index.py --port=8006
directory=/var/www
autorestart=true
redirect_stderr=true

# 启动 Supervisord
(supervisor) [root@tornado-web supervisor]# supervisord  -c /etc/supervisord.conf
(supervisor) [root@tornado-web supervisor]# supervisorctl
hello                            RUNNING   pid 5107, uptime 0:05:04
supervisor> help

default commands (type help <topic>):
=====================================
add    exit      open  reload  restart   start   tail   
avail  fg        pid   remove  shutdown  status  update 
clear  maintail  quit  reread  signal    stop    version

# 配置和运行 Supervisord 后，就可以退出 virtulenv
(supervisor) [root@tornado-web supervisor]# deactivate
```

![](/images/supervisor.png)

**Reference**

+ [Tornado Docs](http://tornado-zh.readthedocs.io/zh/latest/guide/running.html)
+ [Supervisor Docs](http://supervisord.org/)
+ [Nginx 配置详解](http://einverne.github.io/post/2017/10/nginx-conf.html)
