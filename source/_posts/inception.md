---
title: Inception
date: 2018-02-01 20:36:38
categories: MySQL
---
## 安装Inception

```bash
$ yum install bison cmake ncurses-devel openssl-devel libgcc gcc gcc-c++ git   **/ http://ftp.gnu.org/gnu/bison/
$ git clone https://github.com/mysql-inception/inception.git
$ cd inception && mkdir debug
$ cd debug
$ cmake .. -DWITH_DEBUG=OFF \
 	-DWITH_ZLIB=system \
 	-DCMAKE_INSTALL_PREFIX=/var/inception \
 	-DWITH_SSL=bundled \
 	-DCMAKE_BUILD_TYPE=RELEASE \
 	-DWITH_ZLIB=bundled

...
-- Generating done
-- Build files have been written to: /root/inception/debug

$ make && make install
```

<!-- more -->

## 配置Inception

```bash
$ vi /var/inception/bin/inc.cnf

[inception]
general_log=1
general_log_file=/var/inception/log/inception.log
port=6669
socket=/var/inception/inc.socket
character-set-client-handshake=0
character-set-server=utf8

inception_remote_system_user=inc_bk
inception_remote_system_password=Acqua@107
inception_remote_backup_host=10.0.77.19
inception_remote_backup_port=3306
inception_support_charset=utf8mb4
```

## 启动Inception

两种启动方式：

+ /var/inception/bin/Inception --defaults-file=inc.cnf
+ /var/inception/bin/Inception --port=6669

```bash
$ nohup /var/inception/bin/Inception --defaults-file=inc.cnf > /tmp/inception.out 2>&1 &

$ /var/inception/bin/mysql -h 127.0.0.1 -P 6669
mysql> inception get variables;
```

## 线上配置

+ 线上服务器开启 Binlog。
+ binlog_format=row/mined
+ binlog_row_image=full
+ server_id !=0/1
+ Private account for Inception and Specify related privileges(Read-only / Write)：
  + 
`mysql> grant SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, PROCESS, ALTER, SUPER, REPLICATION SLAVE, REPLICATION CLIENT, TRIGGER on *.* to 'winception'@'%' identified by 'Acqua@107';`
+ Primary KEY
+ `DML & DDL` Executed  Separately.

## 使用Inception

python-devel ===> MySQLdb

**inception.py**

```py
#!/usr/bin/python
#-\*-coding: utf-8-\*-
import MySQLdb
sql='/*--user=winception;--password=Acqua@107;--host=10.0.77.17;--execute=1;--port=3306;*/\
inception_magic_start;\
use bsdb;\
DROP TABLE test;\
inception_magic_commit;'
try:
    conn=MySQLdb.connect(host='127.0.0.1',user='',passwd='',port=6669)
    cur=conn.cursor()
    ret=cur.execute(sql)
    result=cur.fetchall()
    num_fields = len(cur.description)
    field_names = [i[0] for i in cur.description]
    print field_names
    for row in result:
        print row[0], "|",row[1],"|",row[2],"|",row[3],"|",row[4],"|",row[5],"|",row[6],"|",row[7],"|",row[8],"|",row[9],"|",row[10]
    cur.close()
    conn.close()
except MySQLdb.Error,e:
     print "Mysql Error %d: %s" % (e.args[0], e.args[1])
```
  
```bash
$ python inception.py

['ID', 'stage', 'errlevel', 'stagestatus', 'errormessage', 'SQL', 'Affected_rows', 'sequence', 'backup_dbname', 'execute_time', 'sqlsha1']
1 | RERUN | 0 | Execute Successfully | None | use bsdb | 0 | '1517555914_20_0' | None | 0.000 | 
2 | EXECUTED | 0 | Execute Successfully
Backup successfully | None | DROP TABLE test | 0 | '1517555914_20_1' | 10_0_77_17_3306_bsdb | 0.020 | 
```

**Inception日志**

```
$ cat /var/inception/log/inception.log
...
180202 15:18:34    16 Query     set autocommit=0

                   16 Query     /*--user=winception;--password=Acqua@107;--host=10.0.77.17;--execute=1;--port=3306;*/inception_magic_start;use bsdb;DROP TABLE test;inception_magic_commit
...
```

**远程(10.0.77.19)备份信息**

```sql
mysql> show databases;
+----------------------+
| Database             |
+----------------------+
| information_schema   |
| 10_0_77_17_3306_bsdb |
| bsdb                 |
| inception            |
| mysql                |
| performance_schema   |
| sys                  |
+----------------------+
7 rows in set (0.00 sec)

mysql> use 10_0_77_17_3306_bsdb;

mysql> show tables;
+------------------------------------+
| Tables_in_10_0_77_17_3306_bsdb     |
+------------------------------------+
| $_$Inception_backup_information$_$ |
| test                               |
+------------------------------------+
2 rows in set (0.00 sec)

mysql> desc test;
+--------------------+-------------+------+-----+---------+----------------+
| Field              | Type        | Null | Key | Default | Extra          |
+--------------------+-------------+------+-----+---------+----------------+
| id                 | bigint(20)  | NO   | PRI | NULL    | auto_increment |
| rollback_statement | mediumtext  | YES  |     | NULL    |                |
| opid_time          | varchar(50) | YES  |     | NULL    |                |
+--------------------+-------------+------+-----+---------+----------------+
3 rows in set (0.00 sec)

mysql> select * from $_$Inception_backup_information$_$;
+-----------------+-------------------+------------------+-----------------+----------------+-----------------+------------+--------+-----------+------+---------------------+-----------+
| opid_time       | start_binlog_file | start_binlog_pos | end_binlog_file | end_binlog_pos | sql_statement   | host       | dbname | tablename | port | time                | type      |
+-----------------+-------------------+------------------+-----------------+----------------+-----------------+------------+--------+-----------+------+---------------------+-----------+
| 1517555914_20_1 |                   |                0 |                 |              0 | DROP TABLE test | 10.0.77.17 | bsdb   | test      | 3306 | 2018-02-02 15:18:34 | DROPTABLE |
+-----------------+-------------------+------------------+-----------------+----------------+-----------------+------------+--------+-----------+------+---------------------+-----------+
1 row in set (0.00 sec)
```

**Reference**
[MySQL 运维内参](https://item.jd.com/12195430.html)
