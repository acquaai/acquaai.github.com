---
title: MHA - MySQL High Available
date: 2018-01-30 16:58:27
categories: MySQL
---
## MasterHA

1. MHA是一个免费的开源软件。
2. 在已启用`MySQL复制`的系统中，无需改变系统和重新设计数据库即可使用MHA。
3. MHA易于实施，使用简单。 

MHA benefits:

+ Automatic fail-over
+ Fast online master switching 

MHA（Master High Availability）是MySQL的高可用解决方案，故障切换0~30s完成，并在最大程度上保证数据一致性。MHA由管理节点（Manager）和数据节点（Node）组成。Manager可以单独部署在一台独立的主机上管理多个master-slave(Application)集群`[推荐]`，也可以部署在一台slave节点上。Node运行在所有MySQL服务器上，Manager定时probe集群中的Master节点，当master故障时，它自动将最新数据的slave提升为新的master，然后所有其它slave重新指向新的master，整个故障切换过程对应用程序完全透明。

自动故障切换过程中，MHA尝试从故障主服务器上保存二进制日志，最大程度保证数据不丢失，但这并不总是可行。如，当主服务器硬件故障或无法通过SSH访问，MHA无法保存二进制日志，只进行故障转移而丢失了部分最新数据。使用MySQL的半同步复制，可在很大程度上降低数据丢失的风险。MHA与半同步复制结合，只要有一个slave已经收到最新的二进制日志，MHA可以将最新的二进制日志应用于其它所有的slave服务器上，以此保证所有节点数据一致性。

<!-- more -->

本实例中，4台主机如下：

<style>
table th:nth-of-type(1) {
    width: 110px;
}
table th:nth-of-type(2) {
    width: 130px;
}
table th:nth-of-type(3) {
    width: 260px;
}
</style>

|     IP     | Hostname |   Roles     |
|   :---:    |  :---:   |   :---      |
| 10.0.77.16 |  manager | MHA Manager |
| 10.0.77.17 |  master  | App1 Master |
| 10.0.77.18 |  slave1  | App1 Slave1、candidate master |
| 10.0.77.19 |  slave2  | App1 Slave2 |

当master执行维护（无主机硬件故障）时，需要完成下面的流程：

1. 确定所有MySQL服务器的binlog都在同一位置（POS），slave的日志没有滞后。
2. 提升slave1为新的master。
3. 将slave2指向新的master（slave1）。
4. 将原master设置为slave1的slave，用来接收维护期间产生的事务日志。
5. 停止原master的服务，执行维护操作（如upgrade等）。
6. 启动原master的服务，等待接收slave1上产生的事务日志。
7. 重新提升master（当前是slave1的slave）为App1的master。
8. 重新将slave2指向App1的master。
9. 再次配置slave1作为master的slave。

执行以上任务需要`master、slave1、slave2`已正确配置复制。MHA 0.57开始支持GTID，MHA在failover时会自动判断是否是`GTID based failover`，它需要满足3个条件：

+ all nodes: gtid_mode =1
+ all nodes: Executed_Gtid_Set `non empty`
+ at least one: Auto_Position = 1

**基于 binlog 和 GTID 复制在 MHA 故障切换时的区别：**

+ Binlog Based
  + 在master宕机后会尝试从自身拷贝binlog并应用。
  + 若`candidate_master`上没有最新的relay log时，它会从拥有最新relay log的`slave`上生成差异的binlog拷贝到candidate_master并应用。
  + 新master追平的日志后（拥有最新日志），继续采用同样的方法将其它slave追平，最后做change master的操作。

`后面的测试过程中manager上的日志完整记录了这一过程。`

+ GTID Based
  + 若`candidate_master`上没有最新的relay log，candidate_master直接连上拥有最新relay log的slave获取并应用。
  + 新maste尝试从`binlog server`上获取缺少的binlog并应用。
  + 新master的数据同步到最新后，其它的slave连上新master并等待数据完成同步。为了加快切换速度，还可以给`masterha_master_switch`传递 `–wait_until_gtid_in_sync = 1`参数，不用等其它slave完成数据同步。
  
当配置的复制是GTID模式时，如果数据库没有执行过一条事务，show slave status \G命令输出中没有Executed_Gtid_Set:信息，那么MHA会认为是非GTID模式。此外，还需要在app1.cnf中配置binlog server。如果不配置，即使在old master SSH可达的情况下，它也不会去save binlog。这里的binlog1既可以设置master为binlog server，也可以设置其他专用的binlog server。
[binlog1]
hostname=10.0.77.17
hostname=10.0.77.18
hostname=10.0.77.19

推荐配置[GTID](https://acquaai.github.io/2017/12/03/gtid-rep/)复制来实施MHA的高可用。

## Implementation of MHA
### Installation
#### Dependencies for MHA

依赖包可以从这里[下载](https://fedoraproject.org/wiki/EPEL)。

```bash
$ wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
$ rpm -ivh epel-release-latest-7.noarch.rpm
$ yum repolist |grep Extra
epel/x86_64           Extra Packages for Enterprise Linux 7 - x86_64      12,254
```

#### MHA Packages

**Manager**

```bash
manager$ yum install -y perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager perl-Time-HiRes

manager$ wget https://github.com/linyue515/mysql-master-ha/raw/master/mha4mysql-manager-0.57-0.el7.noarch.rpm
manager$ wget https://github.com/linyue515/mysql-master-ha/raw/master/mha4mysql-node-0.57-0.el7.noarch.rpm

manager$ rpm -ivh mha4mysql-node-0.57-0.el7.noarch.rpm
manager$ rpm -ivh mha4mysql-manager-0.57-0.el7.noarch.rpm
```

**Node**

```bash
$ yum install -y perl-DBD-MySQL
$ rpm -ivh mha4mysql-node-0.57-0.el7.noarch.rpm
```

### Preparation on Manager
#### application config file

```bash
manager$ mkdir /etc/mha
manager$ vi /etc/mha/app1.cnf

[server default]
# mysql user and password for MHA
user=mha
password=Acqua@107
repl_password=Acqua@107
ssh_user=root
manager_workdir=/var/log/masterha/app1
remote_workdir=/var/log/masterha/app1
master_ip_online_change_script=/etc/mha/master_ip_online_change
master_ip_failover_script=/etc/mha/master_ip_failover

[server1]
hostname=master

[server2]
hostname=slave1
candidate_master=1   

[server3]
hostname=slave2
no_master=1
```

> 不需要指定master，MHA自动检测确定。
> candidate_master和no_master选项只在故障切换中需要，在线切换master时将指定新master主机。

[MHA参数列表详解](http://wubx.net/mha-parameters/)

#### Copy scripts

```bash
manager$ cd /etc/mha
manager$ vi master_ip_failover
```

```perl
#!/usr/bin/env perl

use strict;
use warnings FATAL => 'all';

use Getopt::Long;

my(
    $command, $ssh_user, $orig_master_host, $orig_master_ip,
    $orig_master_port, $new_master_host, $new_master_ip, $new_master_port
);

my $vip = '10.0.77.20/27';
my $key = '1';
my $ssh_start_vip = "/sbin/ifconfig ens160:$key $vip";
my $ssh_stop_vip = "/sbin/ifconfig ens160:$key down";

GetOptions(
    'command=s' => \$command,
    'ssh_user=s' => \$ssh_user,
    'orig_master_host=s' => \$orig_master_host,
    'orig_master_ip=s' => \$orig_master_ip,
    'orig_master_port=i' => \$orig_master_port,
    'new_master_host=s' => \$new_master_host,
    'new_master_ip=s' => \$new_master_ip,
    'new_master_port=i' => \$new_master_port,
);

exit & main();

sub main {

    print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";

    if ($command eq "stop" || $command eq "stopssh") {

        my $exit_code = 1;
        eval {
            print "Disabling the VIP on old master: $orig_master_host \n"; & stop_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn "Got Error: $@\n";
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif($command eq "start") {

        my $exit_code = 10;
        eval {
            print "Enabling the VIP - $vip on the new master - $new_master_host \n"; & start_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn $@;
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif($command eq "status") {
        print "Checking the Status of the script.. OK \n";
        exit 0;
    } else { & usage();
        exit 1;
    }
}

sub start_vip() {
    `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
}
sub stop_vip() {
    return 0 unless($ssh_user);
    `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
}

sub usage {
    print
        "Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
}
```

```bash
manager$ cat master_ip_failover > master_ip_online_change
manager$ chmod 755 master_ip_failover master_ip_online_change 
```

### Environment preparation

+ 为MHA创建专属用户mha

```sql
master> grant all on *.* to 'mha'@'%' identified by  'Acqua@107';
```

+ 因本测试基于[binlog](https://acquaai.github.io/2017/12/02/binlog-rep/)复制，而slave1为候选 master，除了master外，slave1还需要：[`slave2可选`]

  - 修改slave1的配置文件，增加log-bin参数：`log-bin=mysql-bin`
  - 创建复制用户并授权

```sql
slave1> grant replication slave on *.* to 'replicator'@'%' identified by 'Acqua@107';
```

+ 将slave设置为只读模式，考虑到主、备切换，所以不写入my.cnf配置文件

```bash
$ mysql -uroot -p e"set global read_only=1"
```

+ 配置服务器间SSH互通
All Node executed, include `manager, master, slave1, slave2`.Check hosts DNS or `/etc/hosts`.

```bash
$ ssh-keygen -t rsa

master$ ssh-copy-id root@slave1
master$ ssh-copy-id root@slave2
master$ ssh-copy-id root@master
master$ ssh-copy-id root@manager
```

```bash
manager$ /usr/bin/masterha_check_ssh --conf=/etc/mha/app1.cnf 
......
Mon Jan 29 17:17:38 2018 - [info] All SSH connection tests passed successfully.
```

+ 检测复制状态

```bash
manager$ /usr/bin/masterha_check_repl --conf=/etc/mha/app1.cnf
......
MySQL Replication Health is OK.
```

+ Master上增加VIP地址

```bash
master$ ip addr add 10.0.77.20/27 dev ens160:1
```

### Perform fail-over

```bash
manager$ nohup /usr/bin/masterha_manager --conf=/etc/mha/app1.cnf >  /var/log/masterha/app1/app1.log 2>&1 &

manager$ /usr/bin/masterha_check_status --conf=/etc/mha/app1.cnf
app1 (pid:32394) is running(0:PING_OK), master:master
```

> + manager在4次尝试连接master失败后，将自动进行master切换，并停止监控。可以配置`secondary_check_script`来消除网络故障。
+ `shutdown_script`用来确保master已经关闭，防止脑裂。如果想停止master的监控，可以使用`masterha_stop –conf=/etc/mha/app1.cnf`命令。

```bash
......
----- Failover Report -----

app1: MySQL Master failover master(10.0.77.17:3306) to slave1(10.0.77.18:3306) succeeded

Master master(10.0.77.17:3306) is down!

Check MHA Manager logs at manager for details.

Started automated(non-interactive) failover.
Invalidated master IP address on master(10.0.77.17:3306)
The latest slave slave1(10.0.77.18:3306) has all relay logs for recovery.
Selected slave1(10.0.77.18:3306) as a new master.
slave1(10.0.77.18:3306): OK: Applying all logs succeeded.
slave1(10.0.77.18:3306): OK: Activated master IP address.
slave2(10.0.77.19:3306): This host has the latest relay log events.
Generating relay diff files from the latest slave succeeded.
slave2(10.0.77.19:3306): OK: Applying all logs succeeded. Slave started, replicating from slave1(10.0.77.18:3306)
slave1(10.0.77.18:3306): Resetting slave info succeeded.
Master failover to slave1(10.0.77.18:3306) completed successfully.
```

### Manual fail-over

手动切换不需要开启 MHA Manager 监控。

```bash
master$ systemctl stop mysqld   **/模拟master宕机，手动切换
manager$ /usr/bin/masterha_master_switch --master_state=dead --conf=/etc/mha/app1.cnf --dead_master_host=master
```

```bash
......
Master master(10.0.77.17:3306) is dead. Proceed? (yes/NO):
......
----- Failover Report -----

app1: MySQL Master failover master(10.0.77.17:3306) to slave1(10.0.77.18:3306) succeeded

Master master(10.0.77.17:3306) is down!

Check MHA Manager logs at manager for details.

Started manual(interactive) failover.
Invalidated master IP address on master(10.0.77.17:3306)
The latest slave slave1(10.0.77.18:3306) has all relay logs for recovery.
Selected slave1(10.0.77.18:3306) as a new master.
slave1(10.0.77.18:3306): OK: Applying all logs succeeded.
slave1(10.0.77.18:3306): OK: Activated master IP address.
slave2(10.0.77.19:3306): This host has the latest relay log events.
Generating relay diff files from the latest slave succeeded.
slave2(10.0.77.19:3306): OK: Applying all logs succeeded. Slave started, replicating from slave1(10.0.77.18:3306)
slave1(10.0.77.18:3306): Resetting slave info succeeded.
Master failover to slave1(10.0.77.18:3306) completed successfully.
```

### Online Master Switching(Planned Maintenance)

关闭 MHA Manager 监控，完成后开启。

```bash
manager$ /usr/bin/masterha_master_switch --conf=/etc/mha/app1.cnf --master_state=alive --new_master_host=slave1 --orig_master_is_new_slave
```

> + 建议在切换master之前先用masterha_check_ssh和masterha_check_repl脚本检查SSH连接和复制运行状况。
+ MHA在冻结master上"writes"操作后执行`FLUSH TABLES WITH READ LOCK;`命令。
+ `--orig_master_is_new_slave`选项将在切换过程后使用原master成为新master的slave。

```bash
......
From:
master(10.0.77.17:3306) (current master)
 +--slave1(10.0.77.18:3306)
 +--slave2(10.0.77.19:3306)

To:
slave1(10.0.77.18:3306) (new master)
 +--slave2(10.0.77.19:3306)
 +--master(10.0.77.17:3306)

Starting master switch from master(10.0.77.17:3306) to slave1(10.0.77.18:3306)? (yes/NO):
......
Enabling the VIP - 10.0.77.20/27 on the new master - slave1 
Use of uninitialized value $ssh_user in concatenation (.) or string at /etc/mha/master_ip_online_change line 74.
Tue Jan 30 15:37:19 2018 - [warning] Proceeding.
Tue Jan 30 15:37:19 2018 - [info] Setting read_only=0 on slave1(10.0.77.18:3306)..
Tue Jan 30 15:37:19 2018 - [info]  ok.
Tue Jan 30 15:37:19 2018 - [info] 
Tue Jan 30 15:37:19 2018 - [info] * Switching slaves in parallel..
Tue Jan 30 15:37:19 2018 - [info] 
Tue Jan 30 15:37:19 2018 - [info] -- Slave switch on host slave2(10.0.77.19:3306) started, pid: 1125
Tue Jan 30 15:37:19 2018 - [info] 
Tue Jan 30 15:37:20 2018 - [info] Log messages from slave2 ...
Tue Jan 30 15:37:20 2018 - [info] 
Tue Jan 30 15:37:19 2018 - [info]  Waiting to execute all relay logs on slave2(10.0.77.19:3306)..
Tue Jan 30 15:37:19 2018 - [info]  master_pos_wait(mysql-bin.000001:1335) completed on slave2(10.0.77.19:3306). Executed 0 events.
Tue Jan 30 15:37:19 2018 - [info]   done.
Tue Jan 30 15:37:19 2018 - [info]  Resetting slave slave2(10.0.77.19:3306) and starting replication from the new master slave1(10.0.77.18:3306)..
Tue Jan 30 15:37:19 2018 - [info]  Executed CHANGE MASTER.
Tue Jan 30 15:37:19 2018 - [info]  Slave started.
Tue Jan 30 15:37:20 2018 - [info] End of log messages from slave2 ...
Tue Jan 30 15:37:20 2018 - [info] 
Tue Jan 30 15:37:20 2018 - [info] -- Slave switch on host slave2(10.0.77.19:3306) succeeded.
Tue Jan 30 15:37:20 2018 - [info] Unlocking all tables on the orig master:
Tue Jan 30 15:37:20 2018 - [info] Executing UNLOCK TABLES..
Tue Jan 30 15:37:20 2018 - [info]  ok.
Tue Jan 30 15:37:20 2018 - [info] Starting orig master as a new slave..
Tue Jan 30 15:37:20 2018 - [info]  Resetting slave master(10.0.77.17:3306) and starting replication from the new master slave1(10.0.77.18:3306)..
Tue Jan 30 15:37:20 2018 - [info]  Executed CHANGE MASTER.
Tue Jan 30 15:37:20 2018 - [info]  Slave started.
Tue Jan 30 15:37:20 2018 - [info] All new slave servers switched successfully.
Tue Jan 30 15:37:20 2018 - [info] 
Tue Jan 30 15:37:20 2018 - [info] * Phase 5: New master cleanup phase..
Tue Jan 30 15:37:20 2018 - [info] 
Tue Jan 30 15:37:20 2018 - [info]  slave1: Resetting slave info succeeded.
Tue Jan 30 15:37:20 2018 - [info] Switching master to slave1(10.0.77.18:3306) completed successfully.
```

### Recovery Master(crashed)

在**自动切换**后，ori-master在故障修复后，加上运气不错的情况下还保存有故障前的数据，那么可以把该主机配置为 new-master 的 slave。在 MHA 的日志中找到发生自动切换的时间点和以下信息：

```bash
manager$ grep -i "All other slaves should start" /var/log/masterha/app1/app1.log
```

```bash
Tue Jan 30 16:53:00 2018 - [info]  All other slaves should start replication from here. Statement should be: 
CHANGE MASTER TO MASTER_HOST='slave1 or 10.0.77.18', MASTER_PORT=3306, MASTER_LOG_FILE='mysql-bin.000002', MASTER_LOG_POS=595, MASTER_USER='replicator', MASTER_PASSWORD='xxx';
```

**Reference**
[MySQL High Available with MHA](https://mysqlstepbystep.com/2015/06/01/mysql-high-available-with-mha-2/)
[MySQL基于MHA高可用部署篇（GTID模式）](http://www.ywnds.com/?p=8129)
