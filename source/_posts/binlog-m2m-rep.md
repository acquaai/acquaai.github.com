---
title: Binlog -  MySQL Master 2 Master Replication
date: 2017-12-02 09:45:24
categories: MySQL
---
## Install MySQL
[Download MySQL Yum Repository](https://dev.mysql.com/downloads/repo/yum/)

``` bash
$ yum update
$ yum install local mysql57-community-release-el7-11.noarch.rpm
$ yum install -y yum-utils
$ yum repolist all|grep mysql
$ yum-cofnig-manager -h	可能需要重配置Repository

$ yum install mysql-server mysql-client
$ systemctl start mysqld
$ grep "temporary password" /var/log/mysqld.log
$ mysql -uroot -p
```

``` sql
mysql> alter user 'root'@'localhost' identified by 'Acqua@000';
```

``` bash
$ mysql_secure_installation
```

<!-- more -->

## Configure

### `MySQL01: 10.0.88.171`
``` bash
$ mkdir /var/log/mysql
$ chown -R mysql:mysql /var/log/mysql
$ vi /etc/my.cnf

server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
expire_logs_days = 10
max_binlog_size = 100M
bind-address = 10.0.88.171
```

### `MySQL02: 10.0.88.172`
``` bash
$ mkdir /var/log/mysql
$ chown -R mysql:mysql /var/log/mysql
$ vi /etc/my.cnf

server-id = 2
log_bin = /var/log/mysql/mysql-bin.log
expire_logs_days = 10
max_binlog_size = 100M
bind-address = 10.0.88.172
```

### Layout
[MySQL 5.7](https://dev.mysql.com/doc/refman/5.7/en/linux-installation-rpm.html) Installation Layout for Linux RPM Packages from the MySQL Developer Zone
![](/images/m_Layout.png)

## Replication

### Replication Users
`Both Nodes to do`

``` sql
mysql> create user 'replicator'@'%' identified by 'Acqua@000';
mysql> grant replication slave on *.* to 'replicator'@'%';
```
> `%: slave_ip`

### Configure Replication
`MySQL01`

``` sql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      769 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

`MySQL02`

``` sql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      607 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

`配置MySQL01的Master`

``` sql
mysql> stop slave;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> change master to master_host='10.0.88.172',master_port=3306,master_user='replicator',master_password='Acqua@000',master_log_file='mysql-bin.000001', master_log_pos=607;
Query OK, 0 rows affected, 2 warnings (0.04 sec)

mysql> start slave;
Query OK, 0 rows affected (0.00 sec)
```

`配置MySQL02的Master`

``` sql
mysql> stop slave;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> change master to master_host='10.0.88.171',master_port=3306,master_user='replicator',master_password='Acqua@000',master_log_file='mysql-bin.000001', master_log_pos=769;
Query OK, 0 rows affected, 2 warnings (0.04 sec)

mysql> start slave;
Query OK, 0 rows affected (0.00 sec)
```

## Validate

~~bypass~~


