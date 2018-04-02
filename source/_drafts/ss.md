---
title: SS
tags:
---
## Shadowsocks Client

```bash
$ curl "https://bootstrap.pypa.io/get-pip.py" -o "get-pip.py"
$ python get-pip.py

$ pip install --upgrade pip
$ pip install shadowsocks

$ vi /etc/shadowsocks.json
{
  "server":"x.x.x.x",
  "server_port":0,
  "local_address": "127.0.0.1",
  "local_port":0,
  "password":"password",
  "timeout":300,
  "method":"aes-256-cfb",
  "workers": 1
}

$ nohup sslocal -c /etc/shadowsocks.json > /dev/null 2>&1 &
$ echo "nohup sslocal -c /etc/shadowsocks.json > /dev/null 2>&1 &" >> /etc/rc.local
```

## Privoxy

[Privoxy](https://www.privoxy.org/sf-download-mirror/Sources/) to **http(s)**

```bash
$ wget https://www.privoxy.org/sf-download-mirror/Sources/3.0.26%20%28stable%29/privoxy-3.0.26-stable-src.tar.gz
$ tar xzvf

$ useradd privoxy

$ cd privoxy-3.0.26-stable
$ autoheader && autoconf
$ ./configure
$ make && make install

$ vi /usr/local/etc/privoxy/config
...
listen-address 127.0.0.1:8118   #default port: 8118
forward-socks5t / 127.0.0.1:0 . #don't forget .
...

$ privoxy --user privoxy /usr/local/etc/privoxy/config

$ vi /etc/profile
export http_proxy=http://127.0.0.1:8118
export https_proxy=http://127.0.0.1:8118

$ source /etc/profile
$ curl www.google.com
```
