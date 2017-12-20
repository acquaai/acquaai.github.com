---
title: MGR单主复制配置实例
date: 2017-12-17 21:05:30
categories: MySQL
---
[MGR单主复制配置流程](https://acquaai.github.io/2017/12/16/mgr-single-m/)

**Practice**

## Configure hosts file or DNS for all nodes communication

## Configure MySQL configuration my.cnf

## GR Initial

```sql
mysql01:
mysql> install plugin group_replication soname "group_replication.so";
mysql> show plugins;
mysql> set global group_replication_group_name = "a7b884ed-5d97-466a-b676-8ca0466063fd";
mysql> set global group_replication_local_address = "10.0.88.171:24901";

mysql> set sql_log_bin = 0;
mysql> create user 'rpl_user'@'%' identified by 'rpl_password';
mysql> grant replication slave on *.* to rpl_user@'%' identified by 'rpl_password';
mysql> change master to master_user = 'rpl_user',master_password = 'rpl_password'
	for channel 'group_replication_recovery';
mysql> set sql_log_bin = 1;

mysql> set global group_replication_bootstrap_group = on;
mysql> start group replication;
mysql> set global group_replication_bootstrap_group = off;
```
<!-- more -->

> group_replication_bootstrap_group参数只在初始化第一个成员时使用，所以不要将该参数写到配置文件中，并且在初始化完成后设置成 OFF。

## Add member to GR

```sql
mysql02:
mysql> install plugin group_replication soname "group_replication.so";
mysql> show plugins;
mysql> set global group_replication_group_name = "a7b884ed-5d97-466a-b676-8ca0466063fd";
mysql> set global group_replication_local_address = "10.0.88.171:24901";
mysql> set global group_replication_group_seeds='10.0.88.171:24901,10.0.88.172:24902';

mysql> set sql_log_bin = 0;
mysql> create user 'rpl_user'@'%' identified by 'rpl_password';
mysql> grant replication slave on *.* to rpl_user@'%' identified by 'rpl_password';
mysql> change master to master_user = 'rpl_user',master_password = 'rpl_password'
        for channel 'group_replication_recovery';
mysql> set sql_log_bin = 1;
mysql> start group_replication;
```

> (Donor) group_replication_recovery，GR中加入新成员后，要从其它节点上把它加入之前的数据复制过来。这部分数据的复制使用 异步复制 机制。GR需要使用叫 group_replication_recovery 的异步复制通道（Channel）。连接到哪个成员上复制数据，由GR插件随机选择的，因此在每个成员上都要配置。

## Monitor GR
```sql
mysql>  select * from performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| group_replication_applier | 34e0eae2-e208-11e7-a22c-005056885c59 | mysql02     |        3306 | ONLINE       |
| group_replication_applier | d77c2faf-e207-11e7-9e58-0050568851e1 | mysql01     |        3306 | ONLINE       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
2 rows in set (0.00 sec)

mysql> select * from performance_schema.replication_group_member_stats\G
*************************** 1. row ***************************
                      CHANNEL_NAME: group_replication_applier
                           VIEW_ID: 15137519717576448:4
                         MEMBER_ID: d77c2faf-e207-11e7-9e58-0050568851e1
       COUNT_TRANSACTIONS_IN_QUEUE: 0
        COUNT_TRANSACTIONS_CHECKED: 0
          COUNT_CONFLICTS_DETECTED: 0
COUNT_TRANSACTIONS_ROWS_VALIDATING: 0
TRANSACTIONS_COMMITTED_ALL_MEMBERS: a7b884ed-5d97-466a-b676-8ca0466063fd:1-3
    LAST_CONFLICT_FREE_TRANSACTION: 
1 row in set (0.00 sec)
```

## Test GR
```sql
mysql01:
mysql> create database tGR;
Query OK, 1 row affected (0.00 sec)

mysql> use tGR;
Database changed
mysql> create table tGR (id int primary key,name varchar(20));
Query OK, 0 rows affected (0.04 sec)

mysql> insert into tGR (id,name) values (1,'hello');
Query OK, 1 row affected (0.01 sec)

mysql> insert into tGR (id,name) values (2,'MGR');
Query OK, 1 row affected (0.01 sec)

mysql> select * from tGR;
+----+-------+
| id | name  |
+----+-------+
|  1 | hello |
|  2 | MGR   |
+----+-------+
2 rows in set (0.00 sec)
```

```sql
mysql02:
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| tGR                |
+--------------------+
5 rows in set (0.00 sec)

mysql> use tGR;
mysql> select * from tGR;
+----+-------+
| id | name  |
+----+-------+
|  1 | hello |
|  2 | MGR   |
+----+-------+
2 rows in set (0.00 sec)

mysql> show variables like 'super_read_only';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| super_read_only | ON    |
+-----------------+-------+
1 row in set (0.00 sec)

mysql> set global super_read_only = OFF;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into tGR (id,name) values (3,'WOW');
Query OK, 1 row affected (0.04 sec)

mysql> set global super_read_only = ON;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from tGR;
+----+-------+
| id | name  |
+----+-------+
|  1 | hello |
|  2 | MGR   |
|  3 | WOW   |
+----+-------+
3 rows in set (0.00 sec)
```

```sql
mysql01:

mysql> select * from tGR;
+----+-------+
| id | name  |
+----+-------+
|  1 | hello |
|  2 | MGR   |
|  3 | WOW   |
+----+-------+
3 rows in set (0.00 sec)
```

## 完整my.cnf参数配置

```text
[mysqld]
server_id = 1	# need change for other nodes
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

# replication and binlog related options
slave_parallel_type = LOGICAL_CLOCK
slave_parallel_workers = 8
slave_preserve_commit_order = ON
relay_log = relay-log

# group replication pre-requisites & recommendations
log_bin = binlog
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON
log_slave_updates = ON
master_info_repository = TABLE
relay_log_info_repository = TABLE
binlog_checksum = NONE
transaction-write-set-extraction = XXHASH64

# group replication specific options
#plugin-load = group_replication.so
group_replication_group_name = a7b884ed-5d97-466a-b676-8ca0466063fd
group_replication_local_address = '10.0.88.171:24901'	# need change for other nodes
group_replication_group_seeds = '10.0.88.171:24901,10.0.88.172:24902'	# need change for other nodes
```
