---
title: Binlog - MySQL Classic Replication
date: 2017-12-02 09:45:24
categories: MySQL
---
GTID replication is [here](https://acquaai.github.io/2017/12/03/gtid-rep/).

## Install MySQL

[Download MySQL Yum Repository](https://dev.mysql.com/downloads/repo/yum/)
[Using the MySQL Yum Repository](https://dev.mysql.com/doc/mysql-yum-repo-quick-guide/en/)

```
$ yum update
$ yum install -y net-tools yum-utils
$ rpm -ivh mysql57-community-release-el7-11.noarch.rpm
$ yum-config-manager --enable mysql57-community
$ yum repolist enabled | grep mysql
$ yum install mysql-community-server
$ systemctl start mysqld
$ grep "temporary password" /var/log/mysqld.log
$ mysql -uroot -p

mysql> alter user 'root'@'localhost' identified by 'Acqua@107';
mysql> flush privileges;

$ mysql_secure_installation
```
<!-- more -->

## Configure Master

```bash
$ vi /etc/my.cnf

[mysqld]
server-id=1
log-bin=mysql-bin
binlog_format=ROW

$ systemctl restart mysqld

mysql> grant replication slave on *.* to 'replicator'@'%' identified by 'Acqua@107';
```

> %: slave_ip

### Take snapshot

Option:备份数据

```bash
$ mysqldump -u root -p --all-databases --flush-privileges --single-transaction --master-data=2 --flush-logs --triggers --routines --events --hex-blob >/path/to/backupdir/full_backup-$TIMESTAMP.sql

对于MyISAM表，需要省略`--single-transaction`选项，而`--master-data=2`选项会自动打开`--lock-all-tables`。
```

对于新安装的 M/S 服务器（No data），不需要做备份，只需要记录日志的位置。

```sql
mysql> show master status \G
*************************** 1. row ***************************
             File: mysql-bin.000001
         Position: 595
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)
```

## Configure Slave

```bash
$ vi /etc/my.cnf

[mysqld]
server-id=2
relay_log=relay-log
skip-slave-start		**/ useful to make any checks before starting the slave
                           (this way, slave must be started manually after each mysql restart)

$ systemctl restart mysqld
```

**Option:**恢复数据

```sql
$ mysql -u root -p < /path/to/backupdir/full_backup-$TIMESTAMP.sql
```

```bash
从备份文件中获取Master的日志位置信息。

$ head -n 50 /path/to/backupdir/full_backup-$TIMESTAMP.sql|grep "CHANGE MASTER TO"
OR
Master> show master status \G
```

### Set the master information on the slave's

```sql

slavel> change master to \
     master_host='10.0.77.17', \
     master_port=3306, \
     master_user='replicator', \
     master_password='Acqua@107', \
     master_log_file='mysql-bin.000001', \
     master_log_pos=595;
     
slavel> start slave;

slavel> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.0.77.17
                  Master_User: replicator
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 595
               Relay_Log_File: relay-log.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes     
                   ......
```
		    
+ If the Slave_IO_State= connecting .... then make sure that the slave user information is set correctly and there is no firewall restrictions between the two servers (master and slave) this could be checked by connecting to the master's MySQL from the salve server by the replication user (in this example, slave_user_name).
+ If both Slave_IO_Running and Slave_SQL_Running = Yes, then the replication had been set up correctly.
+ If the Slave_SQL_Running = No, check the value of Last_SQL_Error for more details about the SQL error.
+ If you know that error and you want to ignore it, you can execute "SET GLOBAL sql_slave_skip_counter = 1;" on the slave and then start the slave again "START SLAVE;".
+ To restrict all normal users from changing data on the slave - which might break the replication - the option "read-only" should be added in the slave's my.cnf file.
+ the server option "server-id" must be unique among all servers inside the replication (masters and slaves).
+ If your database size is big (100GB or so) Xtrabackup tool could be used instead of mysqldump - when preparing the master snapshot - for faster backup and restore operations. For more information on how to use Xtrabackup, check out this [link](http://fromdual.com/node/835).

## 配置半同步复制

相比传统的binlog异步复制，半同步复制提高了数据完整性，防止 主/从 数据不一致。

+ 所有master、slave服务器需要安装半同步插件

```sql
master> install plugin rpl_semi_sync_master soname 'semisync_master.so';
slave> install plugin rpl_semi_sync_slave soname 'semisync_slave.so';
```

+ master开启半同步

```sql
master> set global rpl_semi_sync_master_enabled = 1;
master> set global rpl_semi_sync_master_timeout = 100;
```

+ slave开启半同步

```sql
slave> set global rpl_semi_sync_slave_enabled = on;
```

+ slave重启IO线程

```sql
slave> stop slave io_thread;
slave> start slave io_thread;
```

**Reference**
http://www.fromdual.com/how_to_setup_mysql_master-slave_replication
