---
title: MGR单主复制配置流程
date: 2017-12-16 15:54:30
categories: MySQL
---
## MySQL的参数设置

`修改MySQL参数文件---/etc/my.cnf`

### 开启Binlog和Relaylog功能
```  
server_id = 1
log_bin = binlog
log_slave_updates = ON
relay_log = relay-log
```

### 开启GTID功能
Group Replication必须使用GTID功能。

```
gtid_mode = ON
enforce_gtid_consistency = ON
```
<!-- more -->

### 设置ROW格式的Binlog
Group Replication只支持ROW格式的Binlog。

```
binlog_format = ROW
```

### 禁用binlog_checksum
Group Replication到`5.7.17`还不支持带Checksum的Binlog Event。

```
binlog_checksum = NONE
```

在`5.7.20`中,默认为CRC32：`在测试时，err.log中提示设置为NONE`

```sql
mysql> show variables like '%binlog_checksum%';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| binlog_checksum | CRC32 |
+-----------------+-------+
```

### 使用系统表来存储Slave的信息
Group Replication要使用到多源复制的功能，多源复制必须将Slave（Channel）的状态信息存储到系统表。

```
master_info_repository = TABLE
relay_log_info_repository = TABLE
```

### 开启并行复制
```
slave_parallel_type = 'LOGICAL_CLOCK'
slave_parallel_workers = 8  线程数
slave_preserve_commit_order = ON
```

### 开启主键信息采集功能
除了使用Binlog之外，Group Replication还需要Server层帮助采集被更新数据的主键信息：XXHASH64、MURMUR32，默认OFF。同一个组内的所有成员必须设置相同的哈希算法。

```
transaction_write_set_extraction = XXHASH64
```

## Group Replication插件的使用
### 加载插件

```sql
mysql> install plugin group_replication soname 'group_replication.so';
```

### 启用插件
命令将本MySQL服务器加入到一个存在的GR组内，或者将它初始化为组内的第一个成员。

```sql
mysql> start group_replication;
```

### 停用插件
命令将本MySQL服务器从一个GR组内移除出去。

```sql
mysql> start group_replication;
```

## Group Replication插件的基本参数设置
### 设置组名
每个GR需要唯一识别的组名。在初始化或加入一个组时，必须设置。且组名使用UUID。UUID用来标记组内所有成员上产生的Binlog Event，任何成员生成的GTID都会使用这个UUID，容易区分Event是哪个组产生的,可以使用 `uuidgen`生成。

```
group_replication_group_name = <uuid>
```

### 设置成员的本地地址
每个成员都要有一个独立的TCP端口，成员之间通过此端口通信。

```
group_replication_local_address = <ip:port>
```

### 设置种子成员地址
当加入一个组时，新成员首先必须要和组内成员进行通信来完成加入过程，因此需要至少`一个`当前组内成员地址。这此成员就是`种子成员`。这个变量配置的是其它成员的group_replication_local_address内容。

```
group_replication_group_seeds = <ip:port,ip:port>
```

### 设置成员IP白名单
GR通过白名单地址列表来控制哪些IP地址的MySQL服务器能够加入到组里来。地址列表可以是具体IP或网段，之间用","分隔。若不配置白名单，GR会自动识别本机网口上配置的私网地址和私网网段，只允许和自己在同一私网的MySQL服务器连接到自己的端口。`配置白名单必须要关闭GR`，提前规划IP。

```
group_replication_whitelist = <ip,net,...>
```

### 将参数写入配置文件
插件的参数只能在插件加载后设置，若设置这些参数到配置文件中，须在参数前加前缀: `loose-`

## Group Replication的数据库用户

`_gr_user@localhost`用户
GR使用此用户执行SQL语句来查询和配置数据库中的一些参数，在加载GR时创建。

`root`用户
检查_gr_user@localhost用户是否存在和创建它时，这个session是由root用户来执行的。在使用GR时，必须要有root用户。

## Group Replication组初始化
### 初始化特有的参数
GR组的初始化是在启用第一个成员时完成。在启用第一个成员时，须使用参数告诉GR插件：它是该组的第一个成员，需要做初始化工作。

```
group_replication_bootstrap_group = ON
```
`注：此参数只在初始化第一个成员时使用，所以不能将此参数写入到配置文件中，且在初始化完成后将该参数设置成 OFF 。`

## 组初始化步骤
Group Replication初始化：

```
install plugin group_replication soname 'group_replication.so';
group_replication_group_name = <uuid>
group_replication_local_address = <ip:port>
```

```
group_replication_bootstrap_group = ON
start group_replication
group_replication_bootstrap_group = OFF
```

**Reference**
[MySQL 运维内参](https://item.jd.com/12195430.html)


```
log_bin=binlog
server_id=1
log_slave_updates = ON
relay_log=relay-log
gtid_mode=ON
enforce_gtid_consistency=ON
binlog_format=ROW
binlog_checksum = NONE
master_info_repository=TABLE
relay_log_info_repository=TABLE

slave_parallel_type= "LOGICAL_CLOCK"
slave_parallel_workers=8
slave_preserve_commit_order=ON

transaction_write_set_extraction = XXHASH64

loose-group_replication_group_name = "38aa1dd3-dffe-4c8d-b517-9ebdd6fae70e"
loose-group_replication_local_address = "10.0.88.171:24901"
loose-group_replication_group_seeds = "10.0.88.171:24901,10.0.88.172:24902"
```



