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

## BACKUP
### Full Backup

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

### Full Backup

未完待续...
