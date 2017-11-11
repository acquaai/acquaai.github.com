---
title: HAProxy + Keepalived Configuration Examples(2)
date: 2017-11-11 22:51:59
categories: DevOps
---
## Keepalived 2 VIPs Configure
### Node1's keepalived.conf file

<!-- more -->

```bash
vrrp_script chk_haproxy {
   script "pkill -0 haproxy"     
   interval 2                    
   weight 2                      
}

vrrp_instance VI_1 {
   interface ens160              
   state MASTER
   virtual_router_id 51          
   priority 100                  
   advert_int 1
   virtual_ipaddress {
       x.x.x.59                 # the virtual IP
   }
   track_script {
       chk_haproxy
   }
}

vrrp_instance VI_2 {
   interface ens160             
   state BACKUP
   virtual_router_id 52          
   priority 99                  
   advert_int 1
   virtual_ipaddress {
       x.x.x.60                 # the virtual IP
   }
   track_script {
       chk_haproxy
   }
}
```

### Node2's keepalived.conf file

```bash
vrrp_script chk_haproxy {
   script "pkill -0 haproxy"      
   interval 2                     
   weight 2                       
}

vrrp_instance VI_1 {
   interface ens160               
   state BACKUP
   virtual_router_id 51          
   priority 99                  
   advert_int 1
   virtual_ipaddress {
       x.x.x.59                  # the virtual IP
   }
   track_script {
       chk_haproxy
   }
}

vrrp_instance VI_2 {
   interface ens160               
   state MASTER
   virtual_router_id 52          
   priority 100                  
   advert_int 1
   virtual_ipaddress {
       x.x.x.60                 # the virtual IP
   }
   track_script {
       chk_haproxy
   }
}
```

## Keepalived vrrp_sync_group Configure

> 应用场景为：如果路由有2个网段，内网和外网，每个网段开启一个VRRP实例，假设VRRP配置为检查内网，当外网出现问题时，VRRPD会认为自己是健康的，则不会发送Master和Backup的切换，从而导致问题，Sync Group可以把两个实例都放入Sync Group，这样Group 里任何一个实例出现问题都会发生切换。

```bash
vrrp_sync_group VG1 {
    group {
        VI_1
        VI_2
    }
}
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.23.8.80
    }
}
vrrp_instance VI_2 {
    state MASTER
    interface eth1
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        172.18.1.254
    }
}
virtual_server 10.23.8.80 80 {
    delay_loop 6
    lb_algo wlc
    lb_kind NAT
    persistence_timeout 600
    protocol TCP
    real_server 172.18.1.11 80 {
        weight 100
        TCP_CHECK {
            connect_timeout 3
        }
    }
    real_server 172.18.1.12 80 {
        weight 100
        TCP_CHECK {
            connect_timeout 3
        }
    }
    real_server 172.18.1.13 80 {
        weight 100
        TCP_CHECK {
            connect_timeout 3
        }
    }
}
```
