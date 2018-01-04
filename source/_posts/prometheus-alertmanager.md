---
title: Prometheus Alerting
date: 2017-12-28 09:55:16
categories: DevOps
---
[Prometheus基本安装、配置参考](https://acquaai.github.io/2017/11/29/Prometheus-Monitoring/)
[官方参考](https://prometheus.io/docs/alerting/alertmanager/)

## 开启prometheus告警规则

```yaml
$ vi prometheus.yml
...
rule_files:
  - /var/prometheus/prometheus-1.7.1.linux-amd64/alert.rules
...
```
<!-- more -->

## 配置alert rules

```yaml
ALERT PingDown
  IF up == 0
  FOR 2m
  LABELS { severity = "warning" }
  ANNOTATIONS {
    summary = "Instance {{ $labels.instance }} down",
    description = "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 2 minutes"
  }

ALERT PingCheck
  IF probe_success{job="PingCheck"} == 0
  FOR 1m
  LABELS {
    severity="warning"
  }
  ANNOTATIONS {
    SUMMARY = "{{ $labels.instance }} down",
    DESCRIPTION = "{{ $labels.instance }}: Node has been down for more than 1 minutes"
  }

ALERT CPUUsage #Windows
  IF (100 - (avg by (instance) (irate(node_cpu{name="node-exporter",mode="idle"}[5m])) * 100)) > 80
  FOR 2m
  LABELS {
    severity="warning"
  }
  ANNOTATIONS {
    SUMMARY = "{{$labels.instance}}: High CPU usage detected",
    DESCRIPTION = "{{$labels.instance}}: CPU usage is above 80% (current value is: {{ $value }})"
  }

ALERT LoadAverage #Linux
  IF ((node_load5 / count without (cpu, mode) (node_cpu{mode="system"})) > 1)
  FOR 2m
  LABELS {
    severity="warning"
  }
  ANNOTATIONS {
    SUMMARY = "{{$labels.instance}}: High LoadAverage detected",
    DESCRIPTION = "{{$labels.instance}}: LoadAverage is high"
  }

ALERT SwapUsage
  IF (((node_memory_SwapTotal-node_memory_SwapFree)/node_memory_SwapTotal)*100) > 75
  FOR 2m
  LABELS {
    severity="warning"
  }
  ANNOTATIONS {
    SUMMARY = "{{$labels.instance}}: Swap usage detected",
    DESCRIPTION = "{{$labels.instance}}: Swap usage usage is above 75% (current value is: {{ $value }})"
  }

ALERT MemoryUsage
  IF (((node_memory_MemTotal-node_memory_MemFree-node_memory_Cached)/(node_memory_MemTotal)*100)) > 75
  FOR 2m
  LABELS {
    severity="warning"
  }
  ANNOTATIONS {
    SUMMARY = "{{$labels.instance}}: High memory usage detected",
    DESCRIPTION = "{{$labels.instance}}: Memory usage is above 75% (current value is: {{ $value }})"
  }

ALERT LowRootDisk
  IF ((node_filesystem_size{mountpoint="/"} - node_filesystem_free{mountpoint="/"} ) / node_filesystem_size{mountpoint="/"} * 100) > 75
  FOR 2m
  LABELS {
    severity="warning"
  }
  ANNOTATIONS {
    SUMMARY = "{{$labels.instance}}: Low root disk space",
    DESCRIPTION = "{{$labels.instance}}: Root disk usage is above 75% (current value is: {{ $value }})"
  }

ALERT HttpCheckDown
  IF probe_success{job="HttpCheck"} == 0
  FOR 3m
  LABELS {
    severity="critical"
  }
  ANNOTATIONS {
    SUMMARY = "{{$labels.instance}}: Http service is down",
    DESCRIPTION = "{{$labels.instance}}: Http request no response in 3 minutes"
  }


ALERT DNSCheck
  IF probe_success{job="DNSCheck"} == 0
  FOR 1m
  LABELS {
    severity="warning"
  }
  ANNOTATIONS {
    SUMMARY = "{{$labels.instance}}: DNS service is down",
    DESCRIPTION = "{{$labels.instance}}: DNS resolution failed in 1 minutes"
  }
```

## 修改prometheus服务项
增加alertmanager模块

```
$ vi /usr/lib/systemd/system/prometheus.service
ExecStart=/var/prometheus/prometheus-1.7.1.linux-amd64/prometheus -config.file=/var/prometheus/prometheus-1.7.1.linux-amd64/prometheus.yml -storage.local.path=/var/lib/prometheus -alertmanager.url=http://prometheus:9093
```

更新配置文件好后，prometheus重新读取，有两种方法：
 - 通过HTTP API向/-/reload发送POST请求，例：
   curl -X POST http://prometheus:9090/-/reload

 - 向prometheus进程发送SIGHUP信号

## configure alertmanager

```yaml
$ vi simple.yml
global:
  resolve_timeout: 5m
route:
  group_by: ['alertname','severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
  receiver: onealert

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname','service','instance']

receivers:
- name: 'onealert'
  webhook_configs:
  - url: 'http://api.110monitor.com/alert/api/event/prometheus/xxx'
    send_resolved: true
```

## 开机启动alertmanager
可配为systemd服务
```
$ crontab -l
@reboot /var/prometheus/blackbox_exporter-0.11.0/blackbox_exporter --config.file=/var/prometheus/blackbox_exporter-0.11.0/blackbox.yml > /tmp/blackbox.log 2>&1 &
@reboot /var/prometheus/alertmanager-0.12.0/alertmanager -config.file=/var/prometheus/alertmanager-0.12.0/simple.yml > /tmp/alertmanager.log 2>&1 &
```
