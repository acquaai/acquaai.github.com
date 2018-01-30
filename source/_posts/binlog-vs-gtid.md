---
title: Binlog VS GTID Replication
date: 2018-01-30 17:20:38
categories: MySQL
---
Temp:


添加半同步插件（master slave1）

master:
mysql>install plugin rpl_semi_sync_master soname 'semisync_master.so';

slave1:
mysql>install plugin rpl_semi_sync_slave soname 'semisync_slave.so';

master:
mysql>set global rpl_semi_sync_master_enabled = 1;
mysql>set global rpl_semi_sync_master_timeout = 100; 

slave1:
mysql>set global rpl_semi_sync_slave_enabled = ON;


在从库上重启IO线程(slave1)
stop slave IO_thread;
start slave IO_thread;
