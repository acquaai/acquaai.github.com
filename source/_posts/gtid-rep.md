---
title: GTID - MySQL Replication
date: 2017-12-03 16:49:28
categories: MySQL
---
Binlog replication is [here](https://acquaai.github.io/2017/12/02/binlog-rep/).

## MySQL GTID
MySQL `5.6.6`开始支持GTID的新特性，但是需要离线配置，从`5.7.6`开始支持在线。

### Enable GTID conditions
* MySQL 5.6
 * gtid_mode = ON (required)
 * enforce_gtid_consistency = ON (required)
 * log_bin = ON (required)
 * log_slave_updates = ON (required)
 
* MySQL 5.7.13 or higher
 * gtid_mode = ON (required)
 * enforce_gtid_consistency = ON (required)
 * log_bin = ON (option -- 高可用切换)
 * log_slave_updates = ON (option -- 高可用切换) 

<!-- more -->

### Offline procedure

1. 停止业务访问（disable all write）
2. 等待 master(s) 复制所有事务到 slaves
3. 停止所有 MySQL 服务器
4. 所有服务器中的 my.cnf 文件中配置 `gtid-mode=ON`
5. 启动所有 MySQL 服务器
6. 开启业务访问

### Online procedure to enable GTID

1. 所有Server上执行，`不能返回有任何警告`

  ``` sql
  (root@localhost)[(none)]> set @@global.enforce_gtid_consistency = warn;
  ```

2. 所有Server上执行

  ``` sql
  (root@localhost)[(none)]> set @@global.enforce_gtid_consistency = on;
  ```

3. 所有Server上执行

  ``` sql
  (root@localhost)[(none)]> set @@global.gtid_mode = off_permissive;
  ```

4. 这一步将产生携带 GTID 信息的日志，推荐先在 slave 上执行，再在master上执行

  ``` sql
  (root@localhost)[(none)]> set @@global.gtid_mode=on_permissive;
  ```

5. 确认所有节点 binlog 复制完毕，`Value 0`

  ``` sql
  (root@localhost)[(none)]> show status like 'ongoing_anonymous_transaction_count';
  +-------------------------------------+-------+
  | Variable_name                       | Value |
  +-------------------------------------+-------+
  | Ongoing_anonymous_transaction_count | 0     |
  +-------------------------------------+-------+
  1 row in set (0.00 sec)

  必要时执行：
  
  (root@localhost)[(none)]> flush logs;
  ```

6. 所有节点启用 gtid_mode

  ``` sql
  (root@localhost)[(none)]> set @@global.gtid_mode=on;
  ```

7. 修改 my.cnf 配置文件

  ``` bash
  $ echo "gtid_mode=on" >> /etc/my.cnf
  $ echo "enforce_gtid_consistency=on" >> /etc/my.cnf
  ```

8. 启用GTID的自动节点复制，如果使用了`multi-source`复制，可以为每一个源配置`channel`

  ``` sql
  (root@localhost)[(none)]> stop slave [FOR CHANNEL 'channel1'];
  Query OK, 0 rows affected (0.00 sec)

  (root@localhost)[(none)]> change master to master_auto_position=1 [FOR CHANNEL 'channel1'];
  Query OK, 0 rows affected (0.01 sec)

  (root@localhost)[(none)]> start slave [FOR CHANNEL 'channel1'];
  Query OK, 0 rows affected (0.00 sec)
  ```
  
  Reference
  [Official Manual](http://dev.mysql.com/doc/refman/5.7/en/replication-mode-change-online-enable-gtids.html)
