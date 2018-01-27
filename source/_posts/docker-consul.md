---
title: Consul服务发现
date: 2018-01-27 14:31:24
categories: DevOps
---
> 服务发现允许某个组件在想要与其它组件交互时，自动找到对方。由于这些应用本身是分布式的，服务发现机制也需要是分布式的。且服务发现作为分布式应用不同组件之间的"胶水"，其本身还需要足够动态、可靠、适应性强，而且可以快速且一致的共享关于这些服务数据。

## Consul

Consul 是使用`Raft`一致性算法来提供确定的写入机制的特殊数据存储器。Consul暴露了键值存储系统和服务分类系统，并提供高可用、高容错能力，保证强一致性。服务可以将自己注册到Consul，并以高可用且分布式的方式共享这些信息。

<!-- more -->

**学习任务**

+ 创建Consul服务的Docker镜像
+ 构建3台运行Docker的宿主机，并在每台上运行一个Consul。
+ 构建服务，并将其注册到Consul，然后从其它服务查询该数据。

### Consul Image

```bash
$ mkdir consul && cd consul
$ vi Dockerfile
$ docker build -t acqua/consul .
```

[Dockerfile](https://github.com/acquaai/dockerjenkins/tree/master/consul)

Consul默认端口

| Port | Function |
| :--- | :--- |
| 53/udp | DNS服务器 |
| 8300 | 服务器使用的RPC |
| 8301+udp | Serf服务器的LAN端口 |
| 8302+udp | Serf服务器的WAN端口  |
| 8400 | 命令行RPC接入点|
| 8500 | HTTP API  |

### Local Test Consul

```bash
$ docker run -p 8500:8500 -p 53:53/udp \
  -h node1 acqua/consul -server -bootstrap

==> WARNING: Bootstrap mode enabled! Do not enable unless necessary
==> Starting Consul agent...
==> Starting Consul agent RPC...
==> Consul agent running!
         Node name: 'node1'
        Datacenter: 'dc1'
            Server: true (bootstrap: true)
       Client Addr: 0.0.0.0 (HTTP: 8500, HTTPS: -1, DNS: 53, RPC: 8400)
      Cluster Addr: 172.17.0.7 (LAN: 8301, WAN: 8302)
    Gossip encrypt: false, RPC-TLS: false, TLS-Incoming: false
             Atlas: <disabled>

==> Log data will now stream in as it occurs:

    2018/01/26 07:31:17 [INFO] raft: Node at 172.17.0.7:8300 [Follower] entering Follower state
    2018/01/26 07:31:17 [INFO] serf: EventMemberJoin: node1 172.17.0.7
    2018/01/26 07:31:17 [INFO] consul: adding LAN server node1 (Addr: 172.17.0.7:8300) (DC: dc1)
    2018/01/26 07:31:17 [INFO] serf: EventMemberJoin: node1.dc1 172.17.0.7
    2018/01/26 07:31:17 [INFO] consul: adding WAN server node1.dc1 (Addr: 172.17.0.7:8300) (DC: dc1)
    2018/01/26 07:31:17 [ERR] agent: failed to sync remote state: No cluster leader
    2018/01/26 07:31:18 [WARN] raft: Heartbeat timeout reached, starting election
    2018/01/26 07:31:18 [INFO] raft: Node at 172.17.0.7:8300 [Candidate] entering Candidate state
    2018/01/26 07:31:18 [INFO] raft: Election won. Tally: 1
    2018/01/26 07:31:18 [INFO] raft: Node at 172.17.0.7:8300 [Leader] entering Leader state
    2018/01/26 07:31:18 [INFO] consul: cluster leadership acquired
    2018/01/26 07:31:18 [INFO] raft: Disabling EnableSingleNode (bootstrap)
    2018/01/26 07:31:18 [INFO] consul: New leader elected: node1
    2018/01/26 07:31:18 [INFO] consul: member 'node1' joined, marking health alive
    2018/01/26 07:31:20 [INFO] agent: Synced service 'consul'
==> Newer Consul version available: 1.0.3
```

`-server`参数告诉consul代理以服务器的模式运行。
`-bootstrap`参数告诉Consul本节点可以执行 Raft leader选举。

> 每个数据中心最多只有1台Consul服务器可以运行在bootstrap模式下。否则，如果有多个可以进行自选举的节点，整个集群无法保证一致性。

### Consul Cluster
#### Conusl Public IP

```bash
host1$ PUBLIC_IP="$(ifconfig ens160 | awk -F ' *|:' '/inet /{print $3}')"
host1$ echo $PUBLIC_IP
10.0.77.22

假定`host1`运行在bootstrap模式下，故需要将host1的PUBLIC_IP（10.0.77.22）添加到host[2~3]。
```

#### User-defined Container DNS

[Configure container DNS in user-defined networks](https://docs.docker.com/engine/userguide/networking/default_network/configure-dns/)

+ 本地Docker的IP地址，以便Consul来解析DNS
+ Google的DNS服务地址，解析其它请求
+ 为Consul查询指定搜索域

#### Start Consul

**HOST1**

```bash
host1$ ip a show docker0
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:bf:5e:6e:d3 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:bfff:fe5e:6ed3/64 scope link 
       valid_lft forever preferred_lft forever
```

```bash
host1$ docker run -d -h $HOSTNAME \
-p 8300:8300      -p 8301:8301 \
-p 8301:8301/udp  -p 8302:8302 \
-p 8302:8302/udp  -p 8400:8400 \
-p 8500:8500      -p 53:53/udp \
--dns=172.17.0.1 \
--dns=8.8.8.8 \
--dns-search=service.consul \
--name host1_agent acqua/consul -server -advertise=$PUBLIC_IP -bootstrap-expect=3
```

`[ERR] agent: failed to sync remote state: No cluster leader`
因为没有其它节点加入集群，没有触发选举行为。

**HOST2**

```bash
host2$ PUBLIC_IP="$(ifconfig ens160 | awk -F ' *|:' '/inet /{print $3}')"
host2$ echo $PUBLIC_IP
10.0.77.16

host2$ JOIN_IP=10.0.77.22
host2$ echo $JOIN_IP
```

```bash
host2$ docker run -d -h $HOSTNAME \
-p 8300:8300      -p 8301:8301 \
-p 8301:8301/udp  -p 8302:8302 \
-p 8302:8302/udp  -p 8400:8400 \
-p 8500:8500      -p 53:53/udp \
--dns=172.17.0.1 \
--dns=8.8.8.8 \
--dns-search=service.consul \
--name host2_agent acqua/consul -server -advertise=$PUBLIC_IP -join=$JOIN_IP
```

**HOST3**

```bash
host3$ PUBLIC_IP="$(ifconfig ens160 | awk -F ' *|:' '/inet /{print $3}')"
host3$ echo $PUBLIC_IP
10.0.77.17

host3$ JOIN_IP=10.0.77.22
host3$ echo $JOIN_IP
```

```bash
host3$ docker run -d -h $HOSTNAME \
-p 8300:8300      -p 8301:8301 \
-p 8301:8301/udp  -p 8302:8302 \
-p 8302:8302/udp  -p 8400:8400 \
-p 8500:8500      -p 53:53/udp \
--dns=172.17.0.1 \
--dns=8.8.8.8 \
--dns-search=service.consul \
--name host3_agent acqua/consul -server -advertise=$PUBLIC_IP -join=$JOIN_IP
```

```
host1_agent log:
2018/01/27 02:55:04 [INFO] serf: EventMemberJoin: host3 10.0.77.17
2018/01/27 02:55:04 [INFO] consul: adding LAN server host3 (Addr: 10.0.77.17:8300) (DC: dc1)
2018/01/27 02:55:04 [INFO] consul: Attempting bootstrap with nodes: [10.0.77.22:8300 10.0.77.16:8300 10.0.77.17:8300]
2018/01/27 02:55:05 [WARN] raft: Heartbeat timeout reached, starting election
2018/01/27 02:55:05 [INFO] raft: Node at 10.0.77.22:8300 [Candidate] entering Candidate state
2018/01/27 02:55:05 [WARN] raft: Remote peer 10.0.77.17:8300 does not have local node 10.0.77.22:8300 as a peer
2018/01/27 02:55:05 [WARN] raft: Remote peer 10.0.77.16:8300 does not have local node 10.0.77.22:8300 as a peer
2018/01/27 02:55:05 [INFO] raft: Election won. Tally: 2
2018/01/27 02:55:05 [INFO] raft: Node at 10.0.77.22:8300 [Leader] entering Leader state
2018/01/27 02:55:05 [INFO] consul: cluster leadership acquired
2018/01/27 02:55:05 [INFO] consul: New leader elected: host1
2018/01/27 02:55:05 [INFO] raft: pipelining replication to peer 10.0.77.17:8300
2018/01/27 02:55:05 [INFO] raft: pipelining replication to peer 10.0.77.16:8300
2018/01/27 02:55:05 [INFO] consul: member 'host1' joined, marking health alive
2018/01/27 02:55:05 [INFO] consul: member 'host2' joined, marking health alive
2018/01/27 02:55:05 [INFO] consul: member 'host3' joined, marking health alive
2018/01/27 02:55:05 [INFO] agent: Synced service 'consul'
```

![](/images/consul.png)

#### Dig Consul DNS

```bash
$ dig @172.17.0.1 consul.service.consul

; <<>> DiG 9.9.4-RedHat-9.9.4-51.el7_4.1 <<>> @172.17.0.1 consul.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 19003
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;consul.service.consul.         IN      A

;; ANSWER SECTION:
consul.service.consul.  0       IN      A       10.0.77.17
consul.service.consul.  0       IN      A       10.0.77.16
consul.service.consul.  0       IN      A       10.0.77.22

;; Query time: 1 msec
;; SERVER: 172.17.0.1#53(172.17.0.1)
;; WHEN: Sat Jan 27 11:04:05 CST 2018
;; MSG SIZE  rcvd: 150
```

## Register Consul

基于uWSGI框架创建一个分布式应用来演示注册服务。
+ 一个Web应用：distributed_app `host[1~2]`。它在启动时启动相关的worker，并将这些程序作为服务注册到Consul。
+ 一个应用客户端：distributed_client `host3`。它从Consul读取与distributed_app相关的信息，并报告当前应用程序的状态和配置。

### Distributed_app Image

```bash
$ mkdir distributed_app && cd distributed_app
$ vi Dockerfile
$ docker build -t acqua/distributed_app .
```

[Dockerfile](https://github.com/acquaai/dockerjenkins/tree/master/consul/distributed_app)

### Distributed_client Image

```bash
$ mkdir distributed_client && cd distributed_client
$ vi Dockerfile
$ docker build -t acqua/distributed_client .
```

[Dockerfile](https://github.com/acquaai/dockerjenkins/tree/master/consul/distributed_client)

### Start App

```bash
host1$ docker run -h $HOSTNAME -d \
--dns=172.17.0.1 \
--dns=8.8.8.8 \
--dns-search=service.consul \
--name host1_distributed acqua/distributed_app
```

```
log:
[consul] service distributed_app registered successfully
```

![](/images/app1.png)

```bash
host2$ docker run -h $HOSTNAME -d \
--dns=172.17.0.1 \
--dns=8.8.8.8 \
--dns-search=service.consul \
--name host2_distributed acqua/distributed_app
```

![](/images/app2.png)

```bash
host3$ docker run -h $HOSTNAME -d \
--dns=172.17.0.1 \
--dns=8.8.8.8 \
--dns-search=service.consul \
--name host3_distributed acqua/distributed_client
```

```
log:
Application distributed_app with element server1 on port 2001 found on node host1 (10.0.77.22).
We can also resolve DNS - distributed_app resolves to 10.0.77.22 and 10.0.77.16.
Application distributed_app with element server2 on port 2002 found on node host1 (10.0.77.22).
We can also resolve DNS - distributed_app resolves to 10.0.77.22 and 10.0.77.16.
```

**Reference**
[The Docker Book](https://github.com/turnbullpress/dockerbook-code)
[Jimmy Song - Docker内置DNS](https://jimmysong.io/posts/docker-embedded-dns/)
