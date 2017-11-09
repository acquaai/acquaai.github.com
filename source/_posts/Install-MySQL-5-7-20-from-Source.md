---
title: Install MySQL 5.7.20 from Source
date: 2017-11-07 11:44:33
categories: MySQL
---
## Prepare system environment
### Mini install CentOS7.3 and RPMS
``` bash
[root@MySQL ~]# yum install cmake gcc gcc-c++ bison libaio-devel ncurses-devel
```

> On Debian/Ubuntu, package name is libncurses5-dev, on Redhat and derivates it is ncurses-devel.CMake Error: your C compiler: "CMAKE_C_COMPILER-NOTFOUND" was not found.

### Create group & user
``` bash
[root@MySQL ~]# groupadd mysql
[root@MySQL ~]# useradd -r -g mysql -s /sbin/nologin mysql
```
<!-- more -->

### Download [MySQL](https://dev.mysql.com/downloads/mysql/) source code
``` bash
[root@MySQL ~]# tar xzvf mysql-boost-5.7.20.tar.gz
[root@MySQL ~]# cd mysql-5.7.20
[root@MySQL mysql-5.7.20]# mkdir debug
[root@MySQL mysql-5.7.20]# cd debug
[root@MySQL debug]#
```

## Pre-Compile, Compile, Install MySQL
### Pre-Compile
``` bash
[root@MySQL debug]# cmake .. -DBUILD_CONFIG=mysql_release \
			     -DINSTALL_LAYOUT=STANDALONE \
		  	     -DCMAKE_BUILD_TYPE=RelWithDebInfo \
		  	     -DENABLE_DTRACE=OFF \
		  	     -DWITH_EMBEDDED_SERVER=OFF \
		  	     -DWITH_INNODB_MEMCACHED=ON \
		  	     -DWITH_SSL=bundled \
		  	     -DWITH_ZLIB=system \
		  	     -DWITH_PAM=ON \
		  	     -DCMAKE_INSTALL_PREFIX=/var/mysql/ \
		  	     -DINSTALL_PLUGINDIR="/var/mysql/lib/plugin" \
		  	     -DDEFAULT_CHARSET=utf8 \
		  	     -DDEFAULT_COLLATION=utf8_general_ci \
		  	     -DWITH_EDITLINE=bundled \
		  	     -DFEATURE_SET=community \
		  	     -DCOMPILATION_COMMENT="MySQL Server (GPL)" \
		  	     -DWITH_DEBUG=OFF \
		  	     -DWITH_BOOST=../boost \
		  	     -DWITH_SYSTEMD=1		**使用 systemd 管理服务**
```
> -- Configuring done
  -- Generating done  

### Compile
``` bash
[root@MySQL debug]# make -j 4			**-j 使用多少线程数进行编译**
```

### Install
``` bash
[root@MySQL debug]# make install
[root@MySQL debug]# ll /var/mysql/
total 64
drwxr-xr-x.  2 root root  4096 Nov  7 15:09 bin
-rw-r--r--.  1 root root 17987 Sep 13 23:48 COPYING
-rw-r--r--.  1 root root 17987 Sep 13 23:48 COPYING-test
drwxr-xr-x.  2 root root    55 Nov  7 15:09 docs
drwxr-xr-x.  3 root root  4096 Nov  7 15:09 include
drwxr-xr-x.  4 root root   172 Nov  7 15:09 lib
drwxr-xr-x.  4 root root    30 Nov  7 15:09 man
drwxr-xr-x. 10 root root  4096 Nov  7 15:09 mysql-test
-rw-r--r--.  1 root root  2478 Sep 13 23:48 README
-rw-r--r--.  1 root root  2478 Sep 13 23:48 README-test
drwxr-xr-x. 28 root root  4096 Nov  7 15:09 share
drwxr-xr-x.  2 root root    90 Nov  7 15:09 support-files
```

### Configure PATH Environment variables
> export PATH=**/var/mysql/bin**:$PATH:$HOME/bin


## Configure MySQL
### MySQL Owner Group
``` bash
[root@MySQL ~]# chown -R mysql:mysql /var/mysql/
```

### 创建MySQL PID默认目录
> **systemd**
> mysql-5.7.20/debug/scripts/mysqld.service文件中, 把默认的pid文件指定到了/var/run/mysqld/目录, 而并没有事先建立该目录，因此要手动建立该目录并把权限赋给mysql用户。

``` bash
[root@MySQL ~]# mkdir -p /var/run/mysqld
[root@MySQL ~]# chown -R mysql:mysql /var/run/mysqld 
[root@MySQL ~]# cp PATH/mysqld.service /usr/lib/systemd/system
```

### Configure **my.cnf** files

``` bash
[root@MySQL ~]# cat /etc/my.cnf
		[mysqld]
		port=3306
		datadir=/var/mysql/data_3306
		log_error=/var/mysql/data_3306/error.log
		basedir=/var/mysql/

[root@MySQL mysql]# chown mysql:mysql /etc/my.cnf
[root@MySQL mysql]# mkdir /var/mysql/data_3306
[root@MySQL mysql]# chown -R mysql:mysql /var/mysql/data_3306/
```

### Initialize MySQL
``` bash
[root@MySQL ~]# /var/mysql/bin/mysqld --defaults-file=/etc/my.cnf --initialize --user=mysql
[root@MySQL ~]# /var/mysql/bin/mysql_ssl_rsa_setup
```
在/var/mysql/data_3306目录中默认数据库生成, 所有的库文件及目录都是通过文件组mysql及用户mysql创建的。

### Start MySQL
``` bash
[root@MySQL ~]# systemctl start mysqld.service	(**systemd**)
OR 命令行启动
[root@MySQL ~]# nohup /var/mysql/bin/mysqld --defaults-file=/etc/my.cnf --user=mysql > /tmp/mysql.log 2>&1 &
```

## MySQL Security Policies
### Password login
``` bash
[root@MySQL mysql]# /var/mysql/bin/mysql -uroot -h127.0.0.1 -P3306
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)
```
> MySQL 5.7版本中调整了安全策略，默认情况下root帐号不像5.6一样没有密码。在5.7中，数据库启动时为root帐号随机生成一个密码，存储在log_error参数指定的文件中：
[Note] A temporary password is generated for root@localhost: **beuwu*irQ35M**

使用该密码登录之后，5.7强制要求修性密码：
``` sql
[root@MySQL mysql]# mysql -uroot -h127.0.0.1 -P3306 -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.7.20
Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show variables like "%sock%"\G
ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.

mysql> alter user 'root'@'localhost' identified by 'password';
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like "%sock%";
+-----------------------------------------+-----------------+
| Variable_name                           | Value           |
+-----------------------------------------+-----------------+
| performance_schema_max_socket_classes   | 10              |
| performance_schema_max_socket_instances | -1              |
| socket                                  | /tmp/mysql.sock |
+-----------------------------------------+-----------------+
3 rows in set (0.00 sec) 

mysql>
```

> MySQL 5.7提供了一种兼容方式，在初始化数据库时可以通过指定参数**--initialize-insecure**来初始化，其行为就与5.6及之前版本一样了。

### Global Security Configuration
``` bash
[root@MySQL ~]# mysql_secure_installation
```

**Reference**
[MySQL 运维内参](https://item.jd.com/12195430.html)
