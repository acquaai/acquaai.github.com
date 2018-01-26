---
title: 构建Docker多容器应用
date: 2018-01-26 10:53:21
categories: DevOps
---
## Cluster Info
| Container    | Function         |
| :----        | :----            |
|1 Node        | Node应用          |
|1 Redis Master|保存和集群化应用状态 |
|2 Redis Slave |集群化应用状态      |
|1 log         |捕获应用日志        |

<!-- more -->

### Node.js Image

```bash
$ mkdir nodejs && cd nodejs
$ mkdir nodeapp && cd nodeapp
$ wget https://raw.githubusercontent.com/jamtur01/dockerbook-code/
master/code/6/node/nodejs/nodeapp/package.json

$ wget https://raw.githubusercontent.com/jamtur01/dockerbook-code/
master/code/6/node/nodejs/nodeapp/server.js

$ cd ..
$ vi Dockerfile
```

[Dockerfile](https://github.com/acquaai/dockerjenkins/blob/master/nodejs/N_Dockerfile)

```bash
$ docker build -t acqua/nodejs .
```

### Redis Base Image

```bash
$ mkdir redis_base && cd redis_base
$ vi Dockerfile
```

[Dockerfile](https://github.com/acquaai/dockerjenkins/blob/master/nodejs/redis_base/Dockerfile)

```bash
$ docker build -t acqua/redis .
```

### Redis Master Image

```bash
$ mkdir redis_master && cd redis_master
$ vi Dockerfile
```

[Dockerfile](https://github.com/acquaai/dockerjenkins/blob/master/nodejs/redis_primary/Dockerfile)

```bash
$ docker build -t acqua/redis_master .
```

### Redis Slave Image

```bash
$ mkdir redis_slave && cd redis_slave
$ vi Dockerfile
```

[Dockerfile](https://github.com/acquaai/dockerjenkins/blob/master/nodejs/redis_replica/Dockerfile)

```bash
$ docker build -t acqua/redis_slave .
```

### Redis Cluster

```bash
$ docker run -d -h redis_master \
 --name redis_master acqua/redis_master

-h 参数设置容器的主机名。-h会覆盖默认行为（默认将容器主机名设置为容器 ID），容器使用 redis_master 作为主机名，并被本地 DNS 服务正确解析。--name参数指定容器名 redis_master。

$ docker logs redis_master
$ docker run -ti --rm --volumes-from redis_master \
 ubuntu cat /var/log/redis/redis-server.log

$ docker run -d -h redis_slave \
  --name redis_slave \
  --link redis_master:redis_master \
  acqua/redis_slave

$ docker run -ti --rm --volumes-from redis_slave \
  ubuntu cat /var/log/redis/redis-slave.log

......
1:S 26 Jan 01:35:54.084 * Ready to accept connections
1:S 26 Jan 01:35:54.084 * Connecting to MASTER redis_master:6379
1:S 26 Jan 01:35:54.085 * MASTER <-> SLAVE sync started
1:S 26 Jan 01:35:54.085 * Non blocking connect for SYNC fired the event.
1:S 26 Jan 01:35:54.085 * Master replied to PING, replication can continue...
1:S 26 Jan 01:35:54.085 * Partial resynchronization not possible (no cached master)
1:S 26 Jan 01:35:54.086 * Full resync from master: fc21c9aef176d604b8764ee9295f521a2656b2ad:0
1:S 26 Jan 01:35:54.190 * MASTER <-> SLAVE sync: receiving 175 bytes from master
1:S 26 Jan 01:35:54.190 * MASTER <-> SLAVE sync: Flushing old data
1:S 26 Jan 01:35:54.190 * MASTER <-> SLAVE sync: Loading DB in memory
1:S 26 Jan 01:35:54.190 * MASTER <-> SLAVE sync: Finished with success
```

**增加 Redis Slave 节点**

```bash
$ docker run -d -h redis_slave2 \
  --name redis_slave2 \
  --link redis_master:redis_master \
  acqua/redis_slave

$ docker run -ti --rm --volumes-from redis_slave2 \
  ubuntu cat /var/log/redis/redis-slave.log
```

### Node Container

```bash
$ docker run -d \
  --name nodeapp -p 3000:3000 \
  --link redis_master:redis_master \
  acqua/nodejs

$ docker logs nodeapp
Listening on port 3000
```

浏览器的会话状态会先被记录到 redis_master 容器，然后同步到 redis_slave、redis_slave2中。

### Capture App Logs

```bash
$ mkdir logstash && cd logstash
$ vi Dockerfile
```

[Dockerfile](https://github.com/acquaai/dockerjenkins/tree/master/nodejs/logstash)

```bash
$ docker build -t acqua/logstash .

$ docker run  -d --name logstash \
  --volumes-from redis_master \
  --volumes-from nodeapp \
  acqua/logstash

$ docker logs -f logstash
```

刷新浏览器 http://x.x.x.x:3000

```
{
       "message" => "::ffff:10.8.106.12 - - [Fri, 26 Jan 2018 02:14:58 GMT] \"GET / HTTP/1.1\" 200 20 \"-\" \"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/604.4.7 (KHTML, like Gecko) Version/11.0.2 Safari/604.4.7\"",
      "@version" => "1",
    "@timestamp" => "2018-01-26T02:14:58.681Z",
          "host" => "fed88e7f73d0",
          "path" => "/var/log/nodeapp/nodeapp.log",
          "type" => "syslog"
}
```

### No SSH 4 Docker

+ 如果服务通过某个网络接口来管理，可以在启动容器时暴露这个接口。
+ 如果服务通过 socket 来管理，可以通过卷公开这个套接字。
+ 如果要给容器发信号（非杀掉容器）：

  `docker kill -s <signal> <container>`
+ `docker exec`

**Reference**
[The Docker Book](https://github.com/turnbullpress/dockerbook-code)
