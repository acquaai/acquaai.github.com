---
title: Install MySQL 5.7 from Source
date: 2017-11-07 11:44:33
categories: MySQL
---
## Source Installation Requirements
* [CMake](http://www.cmake.org): as the build framework on all platforms.
* [make](http://www.gnu.org/software/make/): it's highly recommended that you use GNU make3.75 or higher. 
* A working ANSI C++ compiler. See the description of the FORCE_UNSUPPORTED_COMPILER. option for some guidelines.
* The Boost C++ libraries are required to build MySQL (but not to use it). Boost 1.59.0 must be installed. 
* [ncurses](https://www.gnu.org/software/ncurses/ncurses.html) library.
* Sufficient free memory.
* Perl is needed if you intend to run test scripts.
* [bison](http://www.gnu.org/software/bison/) 2.1 or higher, (Version 1 is no longer supported.)


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
$ tar zxvf mysql-boost-5.7.20.tar.gz
$ cd mysql-5.7.20
$ mkdir bld
$ cd bld
$ cmake .. -DBUILD_CONFIG=mysql_release \
           -DCMAKE_BUILD_TYPE=RelWithDebInfo \
           -DINSTALL_LAYOUT=STANDALONE \
           -DCMAKE_INSTALL_PREFIX=/usr/local/mysql \
           -DINSTALL_PLUGINDIR="/usr/local/mysql/lib/plugin" \
           -DCOMPILATION_COMMENT="MySQL Server (GPL)" \
           -DFEATURE_SET=community \
           -DDEFAULT_CHARSET=utf8 \
           -DDEFAULT_COLLATION=utf8_general_ci \
           -DWITH_EDITLINE=bundled \
           -DWITH_INNODB_MEMCACHED=ON \
           -DWITH_SSL=bundled \
           -DWITH_ZLIB=system \
           -DWITH_DEBUG=OFF \
           -DWITH_BOOST=../boost
               
           -DENABLE_DTRACE=OFF              **在8.0中弃用**
           -DWITH_EMBEDDED_SERVER=OFF       **在5.7.17中弃用，8.0中移除**
```
> -- Configuring done
  -- Generating done 

```bash
$ make -j 4
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

### Next command is optional

```bash
$ cp support-files/mysql.server /etc/init.d/mysql
$ chmod +x /etc/init.d/mysql
$ chkconfig --add mysql
$ chkconfig --level 345 mysql on
```
> 修改mysql.server文件与实际安装相符。

### Change MySQL root password
> MySQL 5.7版本中调整了安全策略，默认情况下root帐号不像5.6一样没有密码。在5.7中，数据库启动时为root帐号随机生成一个密码，存储在log_error参数指定的文件中：

```bash
$ grep "temporary password" /usr/local/mysql/data_3306/error.log
$ mysql -uroot -hlocalhost -p
```

> 使用该密码登录之后，5.7强制要求修性密码，同时5.7提供了一种兼容方式，在初始化数据库时可以通过指定参数–initialize-insecure来初始化，其行为就与5.6及之前版本一样了。

```sql
mysql> alter user 'root'@'localhost' identified by 'password';
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like '%version%';
+-------------------------+--------------------+
| Variable_name           | Value              |
+-------------------------+--------------------+
| innodb_version          | 5.7.20             |
| protocol_version        | 10                 |
| slave_type_conversions  |                    |
| tls_version             | TLSv1,TLSv1.1      |
| version                 | 5.7.20             |
| version_comment         | MySQL Server (GPL) |
| version_compile_machine | x86_64             |
| version_compile_os      | Linux              |
+-------------------------+--------------------+
8 rows in set (0.01 sec)

```

## Global Security Configuration
``` bash
$ mysql_secure_installation
```

**Reference**
[MySQL 运维内参](https://item.jd.com/12195430.html)
