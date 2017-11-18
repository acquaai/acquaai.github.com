---
title: HAProxy + Keepalived Configuration Examples(1)
date: 2017-11-11 22:28:41
categories: DevOps
---
## HAProxy + Keepalived Configure
### HAProxy Configure

<!-- more -->

+ 编辑haproxy.cfg配置文件

```bash
$ sudo vi /etc/haproxy/haproxy.cfg
global
        log     /dev/log         local0
        log     127.0.0.1        local1  err
        maxconn    30000         //每个haproxy进程所接受的最大并发连接数，max:65535
        chroot     /usr/local/haproxy
        pidfile    /var/run/haproxy.pid
        daemon                        //haproxy以守护进程的方式运行在后台
        stats      socket  /usr/local/haproxy/stats

defaults
        mode    http
        log     global
        option  httplog
        option  dontlognull     //如果产生了一个空连接，那这个空连接的日志将不会记录
        option  forwardfor      //如果后端服务器需要获得客户端真实IP,可以从Http Header中获得客户端IP
        option  httpclose     //开启http协议中服务器端关闭功能,每个请求完毕后主动关闭http通道,使得支持长连接，使得会话可以被重用，使得每一个日志记录都会被记录
        option  redispatch
        retries 3
        option  abortonclose    //当haproxy负载很高时,自动结束掉当前队列处理比较久的链接
        timeout http-request    10s
        timeout queue           1m
        timeout connect         10s   //haproxy与后端服务器连接超时时间
        timeout client          1m    //客户端与haproxy连接后,数据传输完毕,不再有数据传输,即非活动连接的超时时间
        timeout server          1m   //haproxy与后端服务器非活动连接的超时时间
        timeout http-keep-alive 10s  //默认新的http请求连接建立的超时时间，时间较短时可以尽快释放出资源，节约资源
        timeout check           10s  //心跳检测超时时间
        maxconn 30000

listen admin_status
        mode    http
        bind    x.x.x.x:1080      #change
        maxconn 10
        option  httplog
        stats   enable
        stats   refresh 30s
        stats   uri     /stats
        stats   auth    admin:admin     #change
        stats   admin   if      TRUE

        errorfile       400     /usr/local/haproxy/errorfiles/400.http
        errorfile       403     /usr/local/haproxy/errorfiles/403.http
        errorfile       408     /usr/local/haproxy/errorfiles/408.http
        errorfile       500     /usr/local/haproxy/errorfiles/500.http
        errorfile       502     /usr/local/haproxy/errorfiles/502.http
        errorfile       503     /usr/local/haproxy/errorfiles/503.http
        errorfile       504     /usr/local/haproxy/errorfiles/504.http

frontend web
        bind    *:80
        mode    http
        log     global
        option  logasap
        default_backend web-servers

backend web-servers
        balance roundrobin
        mode    http
        option  httpchk GET http://www.xxx.cn/newIndex.aspx
        server web1 x.x.x.x:80 maxconn 2048 cookie web1 weight 3 check inter 2s rise 2 fall 3  //weight：代表权重(默认1，最大为265，0则表示不参与负载均衡)
        server web2 x.x.x.x:80 maxconn 2048 cookie web2 weight 3 check inter 2s rise 2 fall 3
        server web3 x.x.x.x:80 maxconn 2048 cookie web3 weight 3 check inter 2s rise 2 fall 3
```

+ 重启HAProxy服务

```bash
$ sudo systemctl restart haproxy
$ sudo systemctl status haproxy
```

### HAProxy支持的负载均衡算法

> + roundrobin：基于权重进行轮叫，在服务器的处理时间保持均匀分布时，这是最平衡、最公平的算法。此算法是动态的，这表示其权重可以在运行时进行调整，不过，在设计上，每个后端服务器仅能最多接受4128个连接；

> + static-rr：基于权重进行轮叫，与roundrobin类似，但是为静态方法，在运行时调整其服务器权重不会生效；不过，其在后端服务器连接数上没有限制；
> + leastconn：新的连接请求被派发至具有最少连接数目的后端服务器；在有着较长时间会话的场景中推荐使用此算法，如LDAP、SQL等，其并不太适用于较短会话的应用层协议，如HTTP；此算法是动态的，可以在运行时调整其权重；
> + source：将请求的源地址进行hash运算，并由后端服务器的权重总数相除后派发至某匹配的服务器；这可以使得同一个客户端IP的请求始终被派发至某特定的服务器；不过，当服务器权重总数发生变化时，如某服务器宕机或添加了新的服务器，许多客户端的请求可能会被派发至与此前请求不同的服务器；常用于负载均衡无cookie功能的基于TCP的协议；其默认为静态，不过也可以使用hash-type修改此特性；
> + uri：对URI的左半部分(“问题”标记之前的部分)或整个URI进行hash运算，并由服务器的总权重相除后派发至某匹配的服务器；这可以使得对同一个URI的请求总是被派发至某特定的服务器，除非服务器的权重总数发生了变化；此算法常用于代理缓存或反病毒代理以提高缓存的命中率；需要注意的是，此算法仅应用于HTTP后端服务器场景；其默认为静态算法，不过也可以使用hash-type修改此特性；
> + url_param：通过为URL指定的参数在每个HTTP GET请求中将会被检索；如果找到了指定的参数且其通过等于号“=”被赋予了一个值，那么此值将被执行hash运算并被服务器的总权重相除后派发至某匹配的服务器；此算法可以通过追踪请求中的用户标识进而确保同一个用户ID的请求将被送往同一个特定的服务器，除非服务器的总权重发生了变化；如果某请求中没有出现指定的参数或其没有有效值，则使用轮叫算法对相应请求进行调度；此算法默认为静态的，不过其也可以使用hash-type修改此特性；
> + hdr()：对于每个HTTP请求，通过指定的HTTP首部将会被检索；如果相应的首部没有出现或其没有有效值，则使用轮叫算法对相应请求进行调度；其有一个可选选项“use_domain_only”，可在指定检索类似Host类的首部时仅计算域名部分以降低hash算法的运算量；此算法默认为静态的，不过其也可以使用hash-type修改此特性；

### Keepalived Configure
+ 编辑keepalived.conf配置文件

```bash
$ sudo vi /etc/keepalived/keepalived.conf
vrrp_script chk_haproxy {
   script "pkill -0 haproxy"     # verify the pid existance
   interval 2                    # check every 2 seconds
   weight 2                      # add 2 points of prio if OK
}

vrrp_instance VI_1 {
   interface ens160              # interface to monitor
   state MASTER
   virtual_router_id 51          # Assign one ID for this route
   priority 100                  // priority 99 on backup
   unicast_src_ip x.x.x.1        // unicast_src_ip x.x.x.2
   unicast_peer {
        x.x.x.2                  // x.x.x.1
    }
   advert_int 1
   authentication {
        auth_type PASS
        auth_pass 1111
    }
   virtual_ipaddress {
       x.x.x.x                   # the virtual IP
   }
   track_script {
       chk_haproxy
   }
}
```

+ 重启Keepalived服务

```bash
$sudo systemctl restart keepalived
$sudo systemctl status keepalived
```
