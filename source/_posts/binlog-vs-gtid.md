---
title: Binlog VS GTID Replication
date: 2018-01-30 17:20:38
categories: MySQL
---
## Replication

+ 异步复制

主库将事务Binlog事件写入Binlog文件中，主库只会通知一下Dump线程发送这些新的Binlog文件后就继续处理提交操作，并不保证任何一个slave都收到了。

+ 全同步复制

主库提交事务后，在所有的slave都必须收到、并应用且提交这些事务后主库才继续后续操作。

+ 半同步复制

介于全异步与全同步复制之间，主库只需等待至少一个slave收到且 Flush Binlog 到 Relay-log 文件即可，master无需等待所有slave反馈，且只是一个收到的反馈，不是已经完全执行并提交的反馈。

MySQL复制有基于binary logs的[Classic Replication](https://acquaai.github.io/2017/12/02/binlog-rep/) 和基于 [Transaction-based Replication](https://acquaai.github.io/2017/12/03/gtid-rep/)的GTID。

<!-- more -->

配置 Replication 时，有2个主要的操作需要做：

 + Skip or ignore a statement that causes the replication to stop.
 + Re-initialize a slave when the Replication is broke and could not be started anymore.

## SKIP OR IGNORE STATEMENT

某些情况下，如忘记将slave设置为`read_only`模式，master插入了已经存在于slave中的数据，或要更新、删除的数据在slave中未存储，会引发主/从冲突复制停止。
在slave上使用`SHOW SLAVE STATUS`命令将输出类似如下信息：

```bash

Last_SQL_Error: Could not execute Write_rows event on table test.t1; Duplicate entry '4' for key 'PRIMARY', Error_code: 1062; handler error HA_ERR_FOUND_DUPP_KEY; the event's master log mysql-bin.000304, end_log_pos 285

Last_SQL_Error: Could not execute Update_rows event on table test.t1; Can't find record in 't1', Error_code: 1032; handler error HA_ERR_KEY_NOT_FOUND; the event's master log mysql-bin.000304, end_log_pos 492

Last_SQL_Error: Could not execute Delete_rows event on table test.t1; Can't find record in 't1', Error_code: 1032; handler error HA_ERR_KEY_NOT_FOUND; the event's master log mysql-bin.000304, end_log_pos 688
```

### SKIP OR IGNORE STATEMENT处理方法
#### Binlog复制

Binlog方式下只需要在slave上跳过该文件的POS继续复制。

```sql
slave> set global sql_slave_skip_counter = 1;
slave> start slave;
```

#### GTID复制

在处理GTID复制中断时，设置 sql_slave_skip_counter 参数是无效的，而是要[产生一个空事务](https://dev.mysql.com/doc/refman/5.7/en/replication-gtids-failover.html#replication-gtids-failover-empty)。

+ 检查问题事务

```sql
slave> show slave status\G
			.
			.
           Retrieved_Gtid_Set: b9b4712a-df64-11e3-b391-60672090eb04:1-7
           Executed_Gtid_Set: 4f6d62ed-df65-11e3-b395-60672090eb04:1,
                              b9b4712a-df64-11e3-b391-60672090eb04:1-6
           Auto_Position: 1
```

**Retrieved_Gtid_Set:** 从master获取的GTIDs
**Executed_Gtid_Set:** slave要执行的GTIDs
**1-7 VS 1-6:** `7`号事务没有执行，导致复制中断

+ 产生空事务

```sql
slave> set gtid_next = 'b9b4712a-df64-11e3-b391-60672090eb04:7';
slave> begin;
slave> cmmmit;
slave> set gtid_next = 'AUTOMATIC';
slave> start slave;
```

>  Executed_Gtid_Set：中，`4f6d62ed-df65-11e3-b395-60672090eb04:1`是本地（slave）执行的GTIDs，不是从master获取的。 `b9b4712a-df64-11e3-b391-60672090eb04:1-6`是从master中获取并执行的GTIDs。（获取master UUID的方法：1、检查master上`Retrieved_Gtid_Set`中的UUID值；2、master> show variables like 'server_uuid';）
故，要使用master的 UUID 来产生空事务，否则问题依然存在，slave不会启动。

在binlog或GTID复制中，slave恢复后，可能需要Percona的[pt-table-checksum](https://www.percona.com/doc/percona-toolkit/2.2/pt-table-checksum.html)和[pt-table-sync](https://www.percona.com/doc/percona-toolkit/2.2/pt-table-sync.html)工具来修复数据不一致的问题。

## RE-INITIALIZE/ RE-BUILD A SLAVE

由于许多原因，我们最终只能重建slave服务器来恢复数据复制。例如，slave停止了一段时间，master清除了slave需要的二进制日志文件，或者重复数据太多pt-table-checksum和pt-table-sync工具无法修复，就必须使用master的备份重新初始化slave，恢复slave的复制。

### RE-INITIALIZE/ RE-BUILD A SLAVE处理方法
#### Binlog复制

类似错误信息：

```
Last_IO_Errno: 1236
Last_IO_Error: Got fatal error 1236 from master when reading data from binary log: 'Could not find first log file name in binary log index file'
```

处理步骤：

+ Backup on master（可用[Xtrabackup](https://acquaai.github.io/2018/01/31/xtrabackup/)代替mysqldump）

```bash
master$ mysqldump -u root -p --all-databases --flush-privileges --single-transaction --master-data=2 --flush-logs --triggers --routines --events --hex-blob >/path/to/backupdir/full_backup-$TIMESTAMP.sql
```

+ Restore on slave

```bash
slave$ mysql -u root -p < /path/to/backupdir/full_backup-$TIMESTAMP.sql
```

+ Get binary log info from backup

```bash
slave$ head -n 50 /path/to/backupdir/full_backup-$TIMESTAMP.sql|grep "CHANGE MASTER TO"

CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000011', MASTER_LOG_POS=120;
```

+ Change master

```sql
slave> change master to MASTER_LOG_FILE='mysql-bin.000011', MASTER_LOG_POS=120;
```

+ Start slave

```sql
slave> start slave;
```

#### GTID复制

类似错误信息：

```
Last_IO_Errno: 1236
Last_IO_Error: Got fatal error 1236 from master when reading data from binary log: 'The slave is connecting using CHANGE MASTER TO MASTER_AUTO_POSITION = 1, but the master has purged binary logs containing GTIDs that the slave requires.'
```

处理步骤：

+ Backup on master

```bash
master$ mysqldump -u root -p --all-databases --flush-privileges --single-transaction --flush-logs --triggers --routines --events --hex-blob >/path/to/backupdir/full_backup-$TIMESTAMP.sql
```

+ Check GTID value from backup

```bash
slave$ head -n 50 /path/to/backupdir/full_backup-$TIMESTAMP.sql|grep PURGED

SET @@GLOBAL.GTID_PURGED='b9b4712a-df64-11e3-b391-60672090eb04:1-8';
```

+ Reset the GTID_EXECUTED and GTID_PURGED values on the slave:

GTID_EXECUTED为只读参数，`reset master`非常重要。使用[Xtrabackup](https://acquaai.github.io/2018/01/31/xtrabackup/)工具备份可以代替reset master操作。

```sql
slave> reset master;
```

+ Restore on slave

```bash
slave$ mysql -u root -p < /path/to/backupdir/full_backup-$TIMESTAMP.sql
```

+ Check GTID_EXEUCTED & GTID_PURGED values

```sql
slave> show variables like'gtid_executed';
+---------------+------------------------------------------+
| Variable_name | Value                                    |
+---------------+------------------------------------------+
| gtid_executed | b9b4712a-df64-11e3-b391-60672090eb04:1-8 |
+---------------+------------------------------------------+

slave> show variables like 'gtid_purged';
+---------------+------------------------------------------+
| Variable_name | Value                                    |
+---------------+------------------------------------------+
| gtid_purged   | b9b4712a-df64-11e3-b391-60672090eb04:1-8 |
+---------------+------------------------------------------+
```

+ Start slave

```sql
slave> start slave;
```

> 虽然使用GTID进行数据复制比Binlog具有更多优势，但在修复复制中断的策略是不一样的，没有谁比谁好，只有更适合。

**Reference**
[FROMDUAL](http://fromdual.com/replication-troubleshooting-classic-vs-gtid)
