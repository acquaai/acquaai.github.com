---
title: Percona XtraBackup
date: 2018-01-31 14:30:42
categories: MySQL
---
XtraBackup工具是由[Percona](https://www.percona.com)开发的免费开源工具，用于执行物理备份和还原操作，比使用 mysqldump 和 mysql 执行逻辑备份和还原要快得多。[The  latest manual document](https://www.percona.com/doc/percona-xtrabackup/2.4/index.html).

## Install XtraBackup

XtraBackup可以选择从 Repositories 和 Source Code 安装。

```bash
$ yum install -y http://www.percona.com/downloads/percona-release/redhat/0.1-4/percona-release-0.1-4.noarch.rpm
$ yum list |grep percona-xtrabackup
$ yum install -y libev

$ yum install -y percona-xtrabackup-24
```

<!-- more -->

## Full Backup
### Create Full Backup

```bash
$ innobackupex --user=root --password='Acqua@107' --no-timestamp ~/full-backup
......
180131 15:11:27 completed OK!

$ ll full-backup/
total 12340
......
-rw-r----- 1 root root       22 Jan 31 15:11 xtrabackup_binlog_info
-rw-r----- 1 root root      113 Jan 31 15:11 xtrabackup_checkpoints
......

$ cat xtrabackup_binlog_info 

mysql-bin.000001        1335
mysql-bin.000001   在备份时创建的二进制日志文件
1335   备份到了二进制日志文件位置 191 (POS)

$ cat xtrabackup_checkpoints 

backup_type = full-backuped
from_lsn = 0
to_lsn = 2551592
last_lsn = 2551601
compact = 0
recover_binlog_info = 0
```

### Prepare Full Backup

```bash
$ innobackupex --user=root --password='Acqua@107' --apply-log ~/full-backup
```

`--apply-log`
Prepare a backup in **BACKUP-DIR** by applying the transaction log file named "xtrabackup_logfile" located in the same directory. Also, create new transaction logs. The InnoDB configuration is read from the file "backup-my.cnf".

### Restore Full Backup

`copy back option`恢复全备份之前，先停止MySQL instance后执行下面操作：
> 恢复目录不能为 非空 (/var/lib/mysql)

```bash
$ systemctl stop mysqld
$ innobackupex --user=root --password='Acqua@107' --copy-back ~/full-backup

$ chown -R mysql:mysql /var/lib/mysql
$ systemctl start mysqld
```

验证数据。

### Prepare Binlog Slave From Full Backup

slvae复制中断从全备份重建

**1.Restore Full Backup**

**2.**

```bash
2.$ cat xtrabackup_binlog_info 
mysql-bin.000001        6225
```
**3.**

```sql
slave> change master to
       MASTER_HOST='master_ip',
       MASTER_PORT=master_port,
       MASTER_USER='user_name',
       MASTER_PASSWORD='password',
       MASTER_LOG_FILE='mysql-bin.000001',
       MASTER_LOG_POS=6225;
       
slave> start slave;
```

### Prepare GTID Slave From Full Backup

Xtrabackup v2.1.0开始支持GTID。
slvae复制中断从全备份重建。

**1.Restore Full Backup**
**2.Check the GTID value of the backup**

```bash
$ cat xtrabackup_binlog_info 
mysql-bin.000027	191		b9b4712a-df64-11e3-b391-60672090eb04:1-12
```

**3.Set the variable GTID_PURGED with the GTID value of the backup**

```bash
sql> set GTID_PURGED="b9b4712a-df64-11e3-b391-60672090eb04:1-12";
```

**4.change master**

```bash
slave> change master to
       MASTER_HOST='master_ip',
       MASTER_PORT=master_port,
       MASTER_USER='user_name',
       MASTER_PASSWORD='password',
       MASTER_AUTO_POSITION = 1;
       
slave> start slave;
```

> + Xtrabackup full backup不能备份单个 database 或 table（MyISAM可以）。
> + 调用Xtrabackup工具的 MySQL用户 至少拥有以下权限: RELOAD, LOCK TABLES and REPLICATION)。
> + 当从slave来扩容一个 全新的slave 时，只需要在备份命令中增加`--slave-info & --safe-slave-backup`选项，然后在全新的slave上恢复，最后使用备份目录下的`xtrabackup_slave_info`文件信息来change master完成操作。
> + Prepare备份命令中，可以使用`--use-memory=xG`选项来作为内部`innodb_buffer_pool_size`的内存大小来完成备份。
> + preparation过程包括以下两部分，在Prepare备份命令中使用`--apply-log`将执行这两部分操作：
    + 应用已提交的事务
    + 回滚未提交的事务

## Incremental Backup

增量备份可以减少备份存储空间，缩短备份时间，但恢复具有依赖关系

### Create incremental backup

**0级备份**
增量备份是在 FULL Backup 基础之上创建的，Full Backup是一个0级备份，所以在增量备份之前需要先做0级备份。

```bash
$ innobackupex --user=root --password='Acqua@107' --no-timestamp ~/i-full-backup
```

**Creating the first incremental backup**

```bash
$ innobackupex --user=root --password='Acqua@107' --no-timestamp --incremental ~/i-full-backup/inc1 --incremental-basedir=~/i-full-backup

......
180201 09:50:54 Executing UNLOCK TABLES
180201 09:50:54 All tables unlocked
180201 09:50:54 [00] Copying ib_buffer_pool to /root/full-backup/inc1/ib_buffer_pool
180201 09:50:54 [00]        ...done
180201 09:50:54 Backup created in directory '/root/full-backup/inc1/'
MySQL binlog position: filename 'mysql-bin.000001', position '6225'
180201 09:50:54 [00] Writing /root/full-backup/inc1/backup-my.cnf
180201 09:50:54 [00]        ...done
180201 09:50:54 [00] Writing /root/full-backup/inc1/xtrabackup_info
180201 09:50:54 [00]        ...done
xtrabackup: Transaction log of lsn (2565316) to (2565325) was copied.
180201 09:50:54 completed OK!
```

```bash
$ cat inc1/xtrabackup_checkpoints 

backup_type = incremental
from_lsn = 2565316
to_lsn = 2565316
last_lsn = 2565325
compact = 0
recover_binlog_info = 0
```

**Creating the second incremental backup**

```bash
$ innobackupex --user=root --password='Acqua@107' --no-timestamp --incremental ~/i-full-backup/inc2 --incremental-basedir=~/i-full-backup/inc1

$ cat ~/full-backup/inc2/xtrabackup_checkpoints

backup_type = incremental
from_lsn = 2565316
to_lsn = 2582066
last_lsn = 2582075
compact = 0
recover_binlog_info = 0
```

### Prepare Incremental Backup

如`Prepare Full Backup`备份中，准备过程由两个步骤组成（应用已提交的事务并回滚未提交的事务），使用--apply-log选项只能完成两个步骤。但在incremental backup中必须单独执行如下操作：

**1.--redo-only选项（追加变更数据到full backup）**

```bash
$ innobackupex --user=root --password='Acqua@107' --apply-log --redo-only ~/i-full-backup
```

**2.在第1个增量备份中应用已提交事务**

需要指定全备份的目录，因为全备份目录中包含已提交的事务，将增量备份的所有变更追加到全备份。

```bash
$ innobackupex --user=root --password='Acqua@107' --apply-log --redo-only ~/i-full-backup --incremental-dir=~/i-full-backup/inc1
```

**3.在第2个增量备份中应用已提交事务**

不需要指定第1个增量备份目录，所有变更已在**`2.(第2步)`**完成。

```bash
$ innobackupex --user=root --password='Acqua@107' --apply-log --redo-only ~/i-full-backup --incremental-dir=~/i-full-backup/inc2
```

**4.回滚未提交事务**

全备份目录 ~/i-full-backup 包含 (full + inc1 + inc2 ）备份，所以不需要在 inc 目录中做回滚。

```bash
$ innobackupex --user=root --password='Acqua@107' --apply-log ~/i-full-backup
```

### Restore Incremental Backup

全备份目录（~/i-full-backup）是恢复增量备份是唯一要恢复的目录（不需要增量inc目录），因为全备份目录包含所有数据，可以按全备份恢复来执行增量备份的恢复。

> + 在增量备份中，Xtrabackup为每个表生产两个文件“.delta”和“.meta”（如test.ibd.delta＆test.ibd.meta），.delta文件大小反映了该表自上次增量备份以来的数据变化量。单个增量备份的准备时间将取决于自上次增量以来的数据变化量。

### Differential Backup

差异备份减少了所需的备份准备步骤/时间，因为差异备份将取代之前的所有增量备份。
如果中间损坏了增量备份，差异备份可充当以前增量备份的备份。

## Partial Backup

前文中说过，Xtrabackup工具不能备份InnoDB存储引擎中单个数据库或表，可以使用工具中的`部分备份`方式来达到像备份 MyISAM 的单库/单表的效果，但有一些限制。

### Create partial backup

```bash
$ innobackupex --user=root --password='Acqua@107' --no-timestamp --databases='sys bsdb.restore' ~/partial-backup
```

--databases选项可以被下面2个选项代替：
--include='db.tbl'.
--tables-file=/path/to/file.txt
在file.txt文件中以每行一个符合命名规则的表名来指定多个表。

### prepare partial backup

```bash
$ innobackupex --user=root --password='Acqua@107' --apply-log --export ~/partial-backup
```

对于Xtrabackup不能备份的InnoDB存储引擎中单个库或表，会提示并创建相应的`.exp`文件在导入（恢复）时使用。
现在部分备份已准备好被恢复。

### Restore partial backup

+ 与恢复全备份和增量备份不同，恢复部分备份时，等恢复主机上MySQL实例不能停止。
+ 在等恢复主机上新建表的结构应与在部分备份源表时的结构相同，然后丢弃表空间。

```
soure> show create table restore;
```

```sql
target> create table bsdb.restore (...) ENGINE=INNODB;
target> alter table bsdb.restore discard tablespace;
```

从备份中复制要恢复的表( .ibd & .exp )文件到相应 DB目录下，并授权

```bash
$ cd ~/partial-backup/bsdb/
$ cp restore.ibd restore.exp  /var/lib/mysql/bsdb/
$ chown -R mysql:mysql /var/lib/mysql/bsdb
```

使用新的表空间

```sql
target> alter table bsdb.restore import tablespace;
```

> + The streaming feature is not available in the partial backup.
+ **Restoring/importing individual tables or databases from a partial backup is not applicable unless the destination server is `Percona Server`**.
+ 在源/目标主机上必须开启 `innodb_file_per_table` 选项。
+ 仅能在使用 Percona server中的目标主机上开启 `innodb_expand_import` 选项。这就是为什么只能在Percona server中恢复部分备份的原因。

**Reference**
[FROMDUAL](http://fromdual.com/node/835)
