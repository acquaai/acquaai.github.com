---
title: Golang
date: 2017-11-29 20:45:53
categories: DevOps
---
[Download](https://golang.org/dl/)
[Config Ref](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/01.2.md)

``` bash
$ tar xzvf go1.8.3.linux-amd64.tar.gz  -C /usr/local/
```

配置环境变量 $HOME/.bash_profile（注：v1.8之前版本配置不同）
``` text
PATH=$PATH:$HOME/bin:/usr/local/go/bin
GOPATH=$HOME/go
export PATH GOPATH
```

``` bash
$ go
```
<!-- more -->

goWorkSpace(GOPATH目录)
      `-- bin`  golang编译可执行文件存放路径，可自动生成。
      `-- pkg`  golang编译的.a中间文件存放路径，可自动生成。
      `-- src`  源码路径。按照golang默认约定，go run，go install等命令的当前工作路径（即在此路径下执行上述命令）。

Go & GFW
you can use go get github.com/golang/tools, it's a mirror from https://godoc.org/golang.org/x/tools, then: 
``` bash
$ go get github.com/golang/tools
$ mkdir -p src/golang.org/x/
$ cp -r src/github.com/golang/tools   src/golang.org/x/
```
