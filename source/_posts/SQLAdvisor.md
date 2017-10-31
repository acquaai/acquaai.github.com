---
title: SQLAdvisor
date: 2017-10-28 19:21:38
categories: MySQL
comments: false
---
> 感谢美团点评DBA团队开源了[SQLAdvisor](https://github.com/meituan-dianping/sqladvisor)。SQLAdvisor是基于MySQL原生态词法解析，结合分析SQL中的where条件、聚合条件、多表Join关系给出索引优化建议的调优工具。

## 安装SQLAdvisor
### Clone源代码
``` bash
$ git clone https://github.com/meituan-dianping/sqladvisor.git
```

### 安装依赖包
``` bash
$ sudo yum install cmake libaio-devel libffi-devel glib2 glib2-devel
$ sduo yum install Percona-Server-shared-56
```
<!-- more -->

>**注意**
+ 可能需要配置percona56的yum源：
$ sudo yum install http://www.percona.com/downloads/percona-release/redhat/0.1-3/percona-release-0.1-3.noarch.rpm
+ 编译sqladvisor时依赖perconaserverclient_r, 因此需要安装Percona-Server-shared-56。可能需要配置软链接：
$ cd /usr/lib64/
$ sudo ln -sf libperconaserverclient_r.so.18 libperconaserverclient_r.so
+ 跟据glib安装的路径，修改**sqladvisor/sqladvisor/CMakeLists.txt**文件中两处include_directories针对**glib**设置的路径。
...
include_directories("/usr/lib64/glib-2.0/include")
include_directories("/usr/include/glib-2.0")
...

### 编译sqlparser
``` bash
$ sudo cmake -DBUILD_CONFIG=mysql_release -DCMAKE_BUILD_TYPE=debug -DCMAKE_INSTALL_PREFIX=/usr/local/sqlparser ./
$ sudo make
$ sudo make install
```
>**注意**
+ DCMAKE_INSTALL_PREFIX为sqlparser库文件和头文件的安装目录，其中lib目录包含库文件libsqlparser.so，include目录包含所需的所有头文件。
+ DCMAKE_INSTALL_PREFIX值尽量不要修改，后面安装依赖这个目录。

### 安装SQLAdvisor源码
``` bash
$ cd /root/sqladvisor/sqladvisor
$ sudo cmake -DCMAKE_BUILD_TYPE=debug ./
$ sudo make
安装成功后会在本路径生成sqladvisor的可执行文件。
```

## 使用SQLAdvisor
### 查看命令用法
``` bash
$ ./sqladvisor --help
```
### 单SQL语句优化
``` bash
$ ./sqladvisor -h 127.0.0.1 -P 3306 -u root -p 'password' -d mysql -q "select * from user" -v 1
2017-10-28 15:45:13 2045 [Note] 第1步: 对SQL解析优化之后得到的SQL:select `*` AS `*` from `mysql`.`user` 
2017-10-28 15:45:13 2045 [Note] 第2步：表user 的SQL太逆天,没有优化建议 
2017-10-28 15:45:13 2045 [Note] 第3步: SQLAdvisor结束! 
```

### 多SQL语句优化
+ 将多条SQL语句定义在一个文件中
``` bash
$ cat sqls.cnf
[sqladvisor]
username=root
password=password
host=127.0.0.1
port=3306
dbname=mysql
sqls=select * from user ; select * from engine_cost ; select * from server_cost
```

+ 执行定义文件
``` bash
$ ./sqladvisor -f sqls.cnf -v 1
2017-10-28 15:57:03 2102 [Note] 第1步: 对SQL解析优化之后得到的SQL:select `*` AS `*` from `mysql`.`user` 
2017-10-28 15:57:03 2102 [Note] 第2步：表user 的SQL太逆天,没有优化建议 
2017-10-28 15:57:03 2102 [Note] 第3步: SQLAdvisor结束! 

2017-10-28 15:57:03 2102 [Note] 第1步: 对SQL解析优化之后得到的SQL:select `*` AS `*` from `mysql`.`engine_cost` 
2017-10-28 15:57:03 2102 [Note] 第2步：表engine_cost 的SQL太逆天,没有优化建议 
2017-10-28 15:57:03 2102 [Note] 第3步: SQLAdvisor结束! 

2017-10-28 15:57:03 2102 [Note] 第1步: 对SQL解析优化之后得到的SQL:select `*` AS `*` from `mysql`.`server_cost` 
2017-10-28 15:57:03 2102 [Note] 第2步：表server_cost 的SQL太逆天,没有优化建议 
2017-10-28 15:57:03 2102 [Note] 第3步: SQLAdvisor结束! 
```
