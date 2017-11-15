---
title: Tengine-jemalloc
tags:
---
## Preconfiguration setup

```bash
$ yum -y update
```

```bash

$ cat /etc/redhat-release
  CentOS Linux release 7.4.1708 (Core)

$ yum install -y gcc gcc-c++ autoconf automake zlib-devel openssl-devel wget net-tools bzip2
```


## Install Components

+ PCRE

> PCRE(Perl Compatible Regular Expressions)是一个Perl库，包括 perl 兼容的正则表达式库。
> nginx rewrite依赖于PCRE库，所以在安装Tengine前一定要先安装PCRE。

```bash
$ cd /usr/local/src/
$ wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.41.tar.gz
$ tar xzvf pcre-8.41.tar.gz
$ cd pcre-8.41
$ ./configure --prefix=/usr/local/pcre
$ make && make install
```

+ OpenSSL

> 启用 tengine 支持 https 的访问请求

```bash
$ wget http://www.openssl.org/source/openssl-1.0.2m.tar.gz
$ tar xzvf openssl-1.0.2m.tar.gz
$ cd openssl-1.0.2m
$ ./config --prefix=/usr/local/openssl
$ make && make install
```

+ Zlib

> 启用 tengine 的压缩

```bash
$ wget http://zlib.net/zlib-1.2.11.tar.gz
$ tar xzvf zlib-1.2.11.tar.gz
$ cd zlib-1.2.11
$ ./configure --prefix=/usr/local/zlib
$ make && make install
```

+ jemalloc
> 优化内存管理

```bash
$ wget https://github.com/jemalloc/jemalloc/releases/download/5.0.1/jemalloc-5.0.1.tar.bz2
$ tar xjvf jemalloc-5.0.1.tar.bz2
$ cd jemalloc-5.0.1
$ ./configure --prefix=/usr/local/jemalloc
$ make && make install
```

+ Tengine
> 

```bash
$ groupadd www-data
$ useradd -r -g www-data -s /sbin/nologin www-data
$ mkdir -p /data/www
$ chmod +w /data/www
$ chown -R www-data:www-data /data/www

$ wget http://tengine.taobao.org/download/tengine-2.2.1.tar.gz
$ tar xzvf tengine-2.2.1.tar.gz
$ cd tengine-2.2.1
$ ./configure --prefix=/usr/local/nginx \
              --user=www-data \
              --group=www-data \
              --with-pcre=/usr/local/src/pcre-8.41 \
              --with-openssl=/usr/local/src/openssl-1.0.2m \
              --with-jemalloc=/usr/local/src/jemalloc-5.0.1 \
              --with-zlib=/usr/local/src/zlib-1.2.11 \
              --with-http_ssl_module \
              --with-http_gzip_static_module \
              --with-http_realip_module \
              --with-http_stub_status_module \
              --with-http_concat_module

$ make && make install
```

> 注意配置的时候 –with-pcre 、–with-openssl、–with-jemalloc、–with-zlib的路径为源文件的路径。

## Configure Nginx
```bash
$ vi /usr/local/nginx/conf/nginx.conf
user www-data www-data;
worker_processes  auto;
error_log  logs/error.log crit;

pid        logs/nginx.pid;

events {
    use epoll;
    worker_connections  1024;
}

 server {
        listen       80;
        server_name  10.0.88.175;

        location / {
            root   /data/www;
            index  index.html index.htm;
        }
}
```

### Test

```bash
$ vi /data/www/index.html

<html>
<head><title>nginx index.html</title></head>
<body>
<h1>index.html</h1>
</body>
</html>

```

```bash
$ /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf -t
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful

$ /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
$  lsof -n | grep jemalloc.  **?**

```

```bash

$ vi /usr/lib/systemd/system/nginx.service
[Unit]
Description=The nginx HTTP and reverse proxy server
After=syslog.target network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf -t 
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target

```

```bash
$ systemctl enable nginx
$ systemctl is-enabled nginx
$ lsof -n | grep jemalloc       ** ? **
```
