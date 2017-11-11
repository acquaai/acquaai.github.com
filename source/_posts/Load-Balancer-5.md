---
title: HAProxy + Keepalived Configuration Examples(3)
date: 2017-11-11 22:56:37
categories: DevOps
---
## 3Nodes HAProxy + Keepalived
### 3Node's haproxy config files are same

<!-- more -->

```bash
$ sudo cat /etc/haproxy/haproxy.cfg 
global
        log        127.0.0.1   local0
        log        127.0.0.1   local1 notice
        maxconn    4096
        chroot     /usr/local/haproxy
        pidfile    /var/run/haproxy.pid
        daemon
        stats      socket  /usr/local/haproxy/stats

defaults
        mode    http
        log     global
        timeout connect         10s
        timeout client          1m
        timeout server          1m

listen  stats
        mode    http
        bind    *:1080
        stats   enable
        stats   refresh 30s
        stats   uri     /stats
        stats   auth    admin:admin
        stats   admin   if      TRUE

frontend k8s-api
        bind x.x.x.170:5000
        mode tcp
        option tcplog
        default_backend k8s-api

backend k8s-api
        mode tcp
        option tcplog
        option tcp-check
        balance roundrobin
        default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
        server k8s1 x.x.x.171:6443 check
        server k8s1 x.x.x.172:6443 check
        server k8s1 x.x.x.173:6443 check
```

### 3Node's keepalived config files are different

```bash
$ sudo cat /etc/keepalived/keepalived.conf
vrrp_script haproxy-check {
    script "killall -0 haproxy"
    interval 2
    weight 20
}

vrrp_instance haproxy-vip {
    state BACKUP
    priority 101
    interface ens32
    virtual_router_id 47
    advert_int 3

    unicast_src_ip x.x.x.171   //node2: unicast_src_ip x.x.x.172   //node3: unicast_src_ip x.x.x.173
    unicast_peer {
        x.x.x.172             // x.x.x.171                        // x.x.x.171
        x.x.x.173             // x.x.x.173                        // x.x.x.172
    }

    virtual_ipaddress {
        x.x.x.170
    }

    track_script {
        haproxy-check weight 20
    }
}
```
