---
title: MySQL Router
date: 2017-12-21 17:04:22
categories: MySQL
---
> MySQL Router是轻量级的中间件，提供数据库访问的Load Balance（负载均衡）、Failover（故障切换）、读写分离等高可用性服务。Router自身的高可用需要借助类Keepalived来实现。

![](/images/router_arch.png)

<!-- more -->

## Download package

[Router](http://dev.mysql.com/downloads/router/)

```
$ tar xzvf mysql-router-2.1.4-linux-glibc2.12-x86-64bit.tar.gz -C /usr/local/
$ cd /usr/local
$ mv mysql-router-2.1.4-linux-glibc2.12-x86-64bit router
$ ll router/
total 0
drwxr-xr-x 2 7161 31415  25 Jun 27 15:29 bin
drwxrwxr-x 2 7161 31415   6 Jun 27 15:29 data
drwxr-xr-x 4 7161 31415  38 Jun 27 15:29 include
drwxr-xr-x 3 7161 31415 156 Jun 27 15:29 lib
drwxrwxr-x 2 7161 31415   6 Jun 27 15:29 run
drwxr-xr-x 3 7161 31415  17 Jun 27 15:29 share
```

## Configure
### MySQL Router Configuration file

```
$ /usr/local/router/share/doc/mysqlrouter/sample_mysqlrouter.conf  /etc/mysqlrouter.conf
$ vi /etc/mysqlrouter.conf
[DEFAULT]
logging_folder =
plugin_folder = /usr/local/router/lib/mysqlrouter
config_folder = /etc
runtime_folder = /var/run

[logger]
level = INFO

[routing:read_only]
bind_address = 192.168.0.170
bind_port = 7002
mode = read-only
destinations = 192.168.0.172:3306

[routing:read_write]
bind_address = 192.168.0.170
bind_port = 7001
mode = read-write
destinations = 192.168.0.171:3306

[keepalive]
interval = 60
```

> `[routing:read_only]` Clients请求被轮询转发到destinations指定的列表，当遇到列表中不可用服务器时，将跳过到下一个可用服务器处理。若列表中无可用服务器时，路由失败。

> `[routing:read_write]` Clients请求始终被转发到destinations列表中第一个可用服务器，当第一个不可用时才会转发到下一个可用服务器。若列表中无可用服务器时，路由失败。

## Start Router

```
$ cd /usr/local/router/bin/
$ ./mysqlrouter --config=/etc/mysqlrouter.conf 
2017-12-21 16:44:43 INFO    [7fef60055700] keepalive started with interval 60
2017-12-21 16:44:43 INFO    [7fef60055700] keepalive
2017-12-21 16:44:43 INFO    [7fef57fff700] [routing:read_write] started: listening on 192.168.0.170:7001; read-write
2017-12-21 16:44:43 INFO    [7fef5f854700] [routing:read_only] started: listening on 192.168.0.170:7002; read-only
2017-12-21 16:45:43 INFO    [7fef60055700] keepalive
2017-12-21 16:46:43 INFO    [7fef60055700] keepalive

$ mysql -h 10.0.88.171 -P7002 -uroot -p -e "show variables like 'hostname';"
Enter password: 
+---------------+---------+
| Variable_name | Value   |
+---------------+---------+
| hostname      | mysql02 |
+---------------+---------+
$ mysql -h 10.0.88.171 -P7001 -uroot -p -e "show variables like 'hostname';"
Enter password: 
+---------------+---------+
| Variable_name | Value   |
+---------------+---------+
| hostname      | mysql01 |
+---------------+---------+
```

**Reference**
[MySQL High Availability](http://mysqlhighavailability.com/setting-up-mysql-router/)
