---
title: Install MySQL 8.0 from Source
date: 2017-11-13 16:12:49
categories: MySQL
---
## Source Installation Requirements
* [CMake](http://www.cmake.org): as the build framework on all platforms.
* [make](http://www.gnu.org/software/make/): it's highly recommended that you use GNU make3.75 or higher.
* MySQL 8.0 source code permits use of C++11 features. To enable a good level of C++11 support across all supported platforms, the following minimum compiler versions apply:
    * GCC: 4.8 or higher
    * The MySQL C API requires a C++ or C99 compiler to compile.
* The Boost C++ libraries are required to build MySQL (but not to use it). The current version of Boost must be installed. 
* [ncurses](https://www.gnu.org/software/ncurses/ncurses.html) library.
* Sufficient free memory.
* Perl is needed if you intend to run test scripts.

### Preconfiguration setup
```bash
$ cat /etc/redhat-release 
  CentOS Linux release 7.3.1611 (Core)
  
$ yum install cmake gcc gcc-c++ bison libaio-devel ncurses-devel openssl openssl-devel
$ groupadd mysql
$ useradd -r -g mysql -s /bin/false mysql
```

<!-- more -->

### Beginning of source-build specific instructions
Download [MySQL](https://dev.mysql.com/downloads/mysql/) source code

```bash
$ tar zxvf mysql-boost-8.0.3-rc.tar.gz
$ cd mysql-8.0.3-rc
$ mkdir bld
$ cd bld
$ cmake .. -DBUILD_CONFIG=mysql_release \
           -DCMAKE_BUILD_TYPE=RelWithDebInfo \
           -DINSTALL_LAYOUT=STANDALONE \
           -DCMAKE_INSTALL_PREFIX=/usr/local/mysql \
           -DINSTALL_PLUGINDIR="/usr/local/mysql/lib/plugin" \
           -DCOMPILATION_COMMENT="MySQL Server (GPL)" \
           -DDEFAULT_CHARSET=utf8mb4 \
           -DDEFAULT_COLLATION=utf8mb4_unicode_ci \
           -DWITH_EDITLINE=bundled \
           -DWITH_INNODB_MEMCACHED=ON \
           -DWITH_SSL=bundled \
           -DWITH_ZLIB=system \
           -DWITH_DEBUG=OFF \
           -DWITH_BOOST=../boost
           
         **************SYSTEMD****************
           -DWITH_SYSTEMD=1
           -DSYSTEMD_PID_DIR=/var/run/mysqld
           -DSYSTEMD_SERVICE_NAME=mysqld
```
> -- Configuring done
  -- Generating done 

```bash
$ make -j 8
$ make install
```

### Configure MySQL System Environment variables
```
export PATH=$PATH:$HOME/bin:/usr/local/mysql/bin
```

### Postinstallation setup

```bash
$ cd /usr/local/mysql
$ mkdir data_3306
$ vi my.cnf
        [mysqld]
        basedir=/usr/local/mysql
        datadir=/usr/local/mysql/data_3306
        socket=/usr/local/mysql/data_3306/mysql.sock
        port=3306
        log_error=/usr/local/mysql/data_3306/error.log
        
        [client]
        socket=/usr/local/mysql/data_3306/mysql.sock

$ chown -R mysql:mysql data_3306
$ chmod -R 750 data_3306
$ chown mysql:mysql my.cnf
$ chmod 750 my.cnf
$ mysqld --defaults-file=/usr/local/mysql/my.cnf --initialize --user=mysql
$ mysql_ssl_rsa_setup --datadir=/usr/local/mysql/data_3306

$ nohup /usr/local/mysql/bin/mysqld --defaults-file=/usr/local/mysql/my.cnf --user=mysql > /tmp/mysql.log 2>&1 &
```

~~Managing MySQL Server with systemd (optional)~~

### Change MySQL root password

```sql
$ grep "temporary password" /usr/local/mysql/data_3306/error.log
$ mysql -uroot -hlocalhost -p

mysql> alter user 'root'@'localhost' identified by 'password';
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like '%version%';
+-------------------------+--------------------+
| Variable_name           | Value              |
+-------------------------+--------------------+
| innodb_version          | 8.0.3              |
| protocol_version        | 10                 |
| slave_type_conversions  |                    |
| tls_version             | TLSv1,TLSv1.1      |
| version                 | 8.0.3-rc-log       |
| version_comment         | MySQL Server (GPL) |
| version_compile_machine | x86_64             |
| version_compile_os      | Linux              |
+-------------------------+--------------------+
8 rows in set (0.01 sec)
```

```bash
$ mysql -uroot -p
mysql: [Warning] Using a password on the command line interface can be insecure.
ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/tmp/mysql.sock' (2)
```

> mysql -uroot做为客户端时，默认会加载/etc/my.cnf配置文件里的[client] sock路径。
> 如果/etc/my.cnf不存在，则会按默认编译里指定的sock来连接, -DMYSQL_UNIX_ADDR
> Default: /tmp/mysql.sock

```bash
$ mysql -uroot -ppassword -S /usr/local/mysql/data_3306/mysql.sock
```

### Shutdown MySQL Server

```bash
$ mysqladmin -uroot -p shutdown
OR:
$ mysqld stop
OR:
$ mysql.server stop
```

**Reference**
[MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/source-installation.html)
