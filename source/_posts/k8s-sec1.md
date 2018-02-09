---
title: Kubernetes Security
date: 2018-02-09 09:27:33
categories: DevOps
---
**Reference**
1. [Docker security](https://docs.docker.com/engine/security/security/)
2. [Docker容器全面的安全防护](http://www.freebuf.com/column/152398.html)

## Docker宿主机安全

+ Minimal Install
 + sudo yum update -y 
 + 按需安装附加包/服务

<!-- more -->

+ SSH Service

**采用认证登录**
认证服务器(C端)生成一对公/私钥，私钥放在C端，公钥上传到S端。
  
```bash
$ ssh-keygen -t rsa

[dockeruser@repo ~]$ mkidr ~/.ssh
[dockeruser@repo ~]$ sudo chmod 700 ~/.ssh

$ scp ~/.ssh/id_rsa.pub dockeruser@10.0.77.16:~/.ssh/authorized_keys
```

**sshd_config**
SSH禁用root登录和使用密码的身份验证。

PermitRootLogin yes => no
PasswordAuthentication yes => no

**使用非` :22 `默认端口**

+ 关闭无用服务/端口

```bash
$ sudo nmap -sU -sS -p 1-65535 localhost

Starting Nmap 6.40 ( http://nmap.org ) at 2018-02-09 09:21 CST
Nmap scan report for localhost (127.0.0.1)
Host is up (0.0000060s latency).
Other addresses for localhost (not scanned): 127.0.0.1
Not shown: 131068 closed ports
PORT     STATE SERVICE
25/tcp   open  smtp
8086/tcp open  d-s-n

Nmap done: 1 IP address (1 host up) scanned in 1.74 seconds

$ sudo systemctl stop postfix
$ sudo yum remove postfix
```

## Docker安全
### API
+ [APIServer认证、授权、准入控制](https://www.kubernetes.org.cn/1995.html)

+ [NetworkPolicy](https://www.kubernetes.org.cn/2781.html)

默认情况下，K8S没有限制集群内Pod间的通信，NetworkPolicy于Pod间创建防火墙。

### 运行参数

 + `-default-ulimit`
限制进程和文件数 -default-ulimit nproc=X:X -default-ulimit nofile=X:X

 + `-icc=false/true`
管理容器间通信，建议设置为 false，使用`--link` 参数允许容器间通信。

 + `-iptables=true`
启用iptables规则
