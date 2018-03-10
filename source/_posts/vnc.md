---
title: VNC Server
date: 2018-03-10 11:20:21
categories: DevOps
---
```bash
$ yum -y install tigervnc-server
$ su - acqua
$ vncpasswd
$ cp /lib/systemd/system/vncserver@.service  /etc/systemd/system/vncserver@:1.service

# port 5900+display
$ vi /etc/systemd/system/vncserver@\:1.service

# Quick HowTo:
# 1. Copy this file to /etc/systemd/system/vncserver@.service
# 2. Replace <USER> with the actual user name and edit vncserver
#    parameters appropriately
#   ("User=<USER>" and "/home/<USER>/.vnc/%H%i.pid")
# 3. Run `systemctl daemon-reload`
# 4. Run `systemctl enable vncserver@:<display>.service`
```

<!-- more -->

```bash
[Unit]
Description=Remote desktop service (VNC)
After=syslog.target network.target

[Service]
Type=forking
User=acqua

ExecStartPre=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'
ExecStart=/sbin/runuser -l acqua -c "/usr/bin/vncserver %i -geometry 1280x1024"
PIDFile=/home/acqua/.vnc/%H%i.pid
ExecStop=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'
[Install]
WantedBy=multi-user.target

$ systemctl daemon-reload
$ systemctl start vncserver@:1
$ systemctl status vncserver@:1
$ systemctl enable vncserver@:1
$ ss -nultp |grep vnc

$ firewall-cmd --add-port=5901/tcp
$ firewall-cmd --add-port=5901/tcp --permanent
```

VNC Viewer -> IP:5901 来访问。
