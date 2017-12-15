---
title: MySQL log maintenance
date: 2017-12-15 15:45:02
categories: MySQL
---
我们可以手动将日志文件中的数据刷到磁盘上，这一动作将触发MySQL服务器使用一个新的日志文件。过程如下：

* If general query logging or slow query logging to a log file is enabled, the server `closes and reopens` the general query log file or slow query log file.

* If binary logging is enabled, the server closes the current binary log file and opens a new log file with the next sequence number.

* If the server was started with the --log-error option to cause the error log to be written to a file, the server closes and reopens the log file.

The server creates a new binary log file when you flush the logs. However, it just closes and reopens the general and slow query log files. To cause new files to be created on Unix, rename the current log files before flushing them. At flush time, the server opens new log files with the original names. For example, if the general and slow query log files are named mysql.log and mysql-slow.log, you can use a series of commands like this:
<!-- more -->

``` sql
root:(none)> show variables like '%slow%';
+---------------------------+---------------------------------+
| Variable_name             | Value                           |
+---------------------------+---------------------------------+
| log_slow_admin_statements | OFF                             |
| log_slow_slave_statements | OFF                             |
| slow_launch_time          | 2                               |
| slow_query_log            | ON                              |
| slow_query_log_file       | /var/lib/mysql/mysql01-slow.log |
+---------------------------+---------------------------------+
5 rows in set (0.00 sec)
```

```
cd /var/lib/mysql/
mv mysql01-slow.log mysql01-slow.log.old
mysqladmin flush-logs -p
ll mysql01-slow.log*
-rw-r----- 1 mysql mysql 183 Dec 15 14:56 mysql01-slow.log
-rw-r----- 1 mysql mysql 183 Dec 15 14:49 mysql01-slow.log.old
```

通过这种方式，我们可以备份/移走旧的日志文件。
如果有错误日志文件，可以使用类似的策略来备份错误日志文件。

还可以通过禁用日志在运行时重命名常规日志或慢查询日志,此方法适用于任何平台，不需要重新启动服务器。

``` sql
SET GLOBAL general_log = 'OFF';
SET GLOBAL slow_query_log = 'OFF';

在禁用日志的情况下，从命令行重命名日志文件; 
set global slow_query_log_file='/var/lib/mysql/slow_queries_new.log';  

然后再次启用日志：
SET GLOBAL general_log = 'ON';
SET GLOBAL slow_query_log = 'ON';
```

`Validate`

``` sql
root:(none)> select sleep(10) as s;
+---+
| s |
+---+
| 0 |
+---+
1 row in set (10.00 sec)
```

```
tail -5 /var/lib/mysql/mysql01-slow.log
# Time: 2017-12-15T07:38:51.519294Z
# User@Host: root[root] @ localhost []  Id:     5
# Query_time: 10.000416  Lock_time: 0.000000 Rows_sent: 1  Rows_examined: 0
SET timestamp=1513323531;
select sleep(10) as s;
```

Reference
[MySQL 5.7 Reference Manual](https://dev.mysql.com/doc/refman/5.7/en/log-file-maintenance.html)
