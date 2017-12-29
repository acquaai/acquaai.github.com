---
title: Prometheus Monitoring
date: 2017-11-29 20:20:04
categories: DevOps
---
## Prometheus简介
> [Prometheus](https://prometheus.io)是一套开源的监控&报警&时间序列数据库的组合，源于 Google Borgmon 的一个开源监控系统，用 Golang 开发，被称为下一代监控系统。Prometheus 基本原理是通过 HTTP 协议周期性抓取被监控组件的状态，这样做的好处是任意组件只要提供 HTTP 接口就可以接入监控系统，不需要任何 SDK 或者其他的集成过程。这样做非常适合虚拟化环境比如 VM 或者 Docker 。Prometheus 应该是为数不多的适合 Docker、Mesos 、Kubernetes 环境的监控系统之一。输出被监控组件信息的 HTTP 接口被叫做 exporter 。

### Prometheus 的特性
- 自定义多维度的数据模型
- 非常高效的存储，平均一个采样数据占 ~3.5 bytes左右，320万的时间序列，每30秒采样，保持60天，消耗磁盘大概228G。
- 强大的查询语句
- 轻松实现数据可视化

<!-- more -->

`Prometheus daemon` 负责定时去目标上抓取 metrics(指标) 数据，每个抓取目标需要暴露一个http服务的接口给它定时抓取。支持通过配置文件、文本文件、zookeeper、Consul、DNS SRV lookup等方式指定抓取目标。
`Alertmanager` 是独立于Prometheus的一个组件，可以支持Prometheus的查询语句，提供十分灵活的报警方式。
Prometheus支持很多方式的图表可视化，例如十分精美的Grafana，自带的Promdash，以及自身提供的模版引擎等等，还提供HTTP API的查询方式，自定义所需要的输出。
`PushGateway`这个组件是支持Client主动推送 metrics 到PushGateway，而Prometheus只是定时去Gateway上抓取数据。
如果有使用过statsd的用户，则会觉得这十分相似，只是statsd是直接发送给服务器端，而Prometheus主要还是靠进程主动去抓取。

### Prometheus 的数据模型
Prometheus 从根本上所有的存储都是按时间序列去实现的，相同的 metrics(指标名称) 和 label(一个或多个标签) 组成一条时间序列，不同的label表示不同的时间序列。为了支持一些查询，有时还会临时产生一些时间序列存储。

## Prometheus安装和配置
[Download](https://github.com/prometheus/prometheus/releases)

``` bash
$ sudo groupadd prometheus
$ sudo useradd -g prometheus -d /var/lib/prometheus -s /sbin/nologin prometheus
$ sudo tar xvfz prometheus-1.7.1.linux-amd64.tar.gz  -C /usr/local/
$ cd /usr/local
$ mv prometheus-1.7.1.linux-amd64 prometheus
```

### 配置Prometheus监控自已
Prometheus 通过默认 `http://localhost:9090/metrics`  HTTP接口暴露了自己的性能指标数据，当然也就可以配置抓取目标 targets了。Prometheus 采集自身性能数据就是一个十分好的例子了，编辑安装目录下的prometheus.yml配置文件：

``` yaml
global: 
    scrape_interval: 15s 
    scrape_timeout: 10s
    evaluation_interval: 15s 
    external_labels: 
        monitor: codelab-monitor
scrape_configs: 
  -  job_name: prometheus
     scrape_interval: 5s
     scrape_timeout: 5s
     metrics_path: /metrics
     scheme: http
     static_configs:    
      -  targets:
         - 192.168.100.55:9090
```

### 添加系统服务

```bash
 $sudo vi /usr/lib/systemd/system/prometheus.service
 [Unit]
 Description=prometheus
 After=network.target
 [Service]
 Type=simple
 User=root
 ExecStart=/usr/local/prometheus -config.file=/usr/local/prometheus/prometheus.yml -storage.local.path=/var/lib/prometheus
 Restart=on-failure
 
 [Install]
 WantedBy=multi-user.target
```

## Grafana简介
> Grafana 为开源软件，是功能齐全的度量仪表盘和图形编辑器，支持 [Graphite](http://www.oschina.net/p/graphite)、[InfluxDB](http://www.oschina.net/p/influxdb)和 [OpenTSDB](http://www.oschina.net/p/opentsdb)。

### Grafana 主要特性

- 灵活丰富的图形化选项
- 可以混合多种风格
- 支持白天和夜间模式
- 多个数据源
- Graphite 和 InfluxDB 查询编辑器

### Grafana安装和配置
[Download](https://grafana.com/grafana/download)

``` bash
$sudo rpm -ivh grafana-4.3.2-1.x86_64.rpm 
$sudo /bin/systemctl daemon-reload
$sudo /bin/systemctl enable grafana-server.service
$sudo /bin/systemctl start grafana-server.service
```

`http://192.168.100.55:3000`


## 监控Windows主机
[agent](https://github.com/martinlindhe/wmi_exporter)
[dashboards](https://grafana.com/dashboards/2129)

* wmi_exporter为可执行文件，安装后以系统服务的形式存在。
* wmi_exporter能收集的指标项与默认收集项，可根据实际需求，指定启动参数进行收集。

### prometheus.yml文件中增加被监控主机信息

``` yaml
  - job_name: 'win-exporter'
    scrape_interval: 5s
    static_configs:
       - targets:['10.0.88.65:9182','10.0.88.69:9182']
         labels:
           group: 'Web'
```
  
## 监控Linux主机
[agent](https://github.com/prometheus/node_exporter)
[dashboards](https://grafana.com/dashboards/22)

### 编译node_exporter

``` bash
$ sudo go get github.com/prometheus/node_exporter
$ cd ${GOPATH-$HOME/go}/src/github.com/prometheus/node_exporter
$ make
$ nohup ./node_exporter > /tmp/node_exporter.log 2>&1 &
```

### prometheus.yml文件中增加被监控主机信息

``` yaml
   scrape_configs:
     - job_name: 'linux'
         static_configs:
           - targets: ['x.x.x.x:9100']
                labels: instance: node1
```

## blackbox_exporter监控http服务

### prometheus.yml文件中配置信息

``` yaml
- job_name: 'HttpCheck'
    metrics_path: /probe
    params:
      module: [http_2xx]  # Look for a HTTP 200 response.
    metrics_path: /probe
    static_configs:
      - targets:
        - http://www.hnyongxiong.com
        - http://wcf.yubang168.cn
        - http://www.yubang168.cn
        - http://email.yubang168.cn:8080/ybemail/security/login.do
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: x.x.x.x:9115
```

## 监控SQL Server数据库

SQL Server指标采集使用telegraf，而telegraf采用另一个时序数据库[InfluxDB](https://www.influxdata.com/)，它的采集精度更高，为 ms 级，Prometheus为 s 级。InfluxDB安装方法与Prometheus相同，不再赘述。
[agent](https://github.com/influxdata/telegraf)
[dashboards](https://grafana.com/dashboards/409)


### InfluxDB配置文件部分内容

``` bash
$ sudo vi /var/influxdb/etc/influxdb/influxdb.conf
```

``` text
[meta]
  dir = "/var/lib/influxdb/meta"
[data]
  dir = "/var/lib/influxdb/data"
  wal-dir = "/var/lib/influxdb/wal"
[admin]
   enabled = true 
[http]
   enabled = true
   bind-address = "192.168.100.55:8086"
   realm = "InfluxDB"
   log-enabled = true
   max-connection-limit = 0
```

### SQL Server数据采集器

下载 telegraf-1.3.2_windows_amd64.zip 文件，解压得到 telegraf.exe 和 telegraf.conf 文件。

### telegraf.conf配置文件部分内容

``` text
[[inputs.sqlserver]]
   servers = ["Server=10.7.252.56;Port=1433;User Id=telegraf;Password=mystrongpassword;app name=telegraf;log=1;"]
```

### SQL Server创建采集用户
``` sql 
USE master; 
GO
CREATE LOGIN [telegraf] WITH PASSWORD = N'mystrongpassword';
GO
GRANT VIEW SERVER STATE TO [telegraf]; 
GO
GRANT VIEW ANY DEFINITION TO [telegraf]; 
GO
```

### 执行采集
``` bash
c:\telegraf> .\telegraf.exe -config .\telegraf.conf
```
