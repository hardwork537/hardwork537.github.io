---
layout:     post
title:      supervisor
category:   tool
tags:       [supervisor]
description: 进程托管工具supervisor的安装与配置
---

最近在开发一个新服务，本打算用docker部署上去进行管理；不过在跟运维同学沟通后，因为一些特性，用不了docker，只能在想办法了。如果直接启动服务裸跑的话，挂了怎么办？需要找一个可以托管服务的工具，就想到了用supervisor来进行进程管理。由于平时主要使用Mac电脑来开发，所以分享下mac下supervisor相关知识

## 简介

supervisor是用Python开发的一个client/server服务，是Linux/Unix系统下的一个进程管理工具。可以很方便的监听、启动、停止、重启一个或多个进程。用supervisor管理的进程，当一个进程意外被杀死，supervisor监听到进程死后，会自动将它重启，很方便的做到进程自动恢复的功能，不再需要自己写shell脚本来控制。


## 安装

直接用brew进行安装就可以了

```
brew install supervisor
```

通过brew默认安装目录在: ```/usr/local/Cellar/supervisor``` ，当前版本是 4.2.2


## 启动

```
sudo /usr/local/Cellar/supervisor/4.2.2_1/bin/supervisord -c /usr/local/etc/supervisord.conf
```

## 配置文件

从启动命令可以看到，我们用到的配置文件是 ```/usr/local/etc/supervisord.conf```，这也是通过brew安装后默认生成的配置文件

通常情况下，我们只需要修改最后一行就可以

```
169 [include]
170 files = /usr/local/etc/supervisor.d/*.ini
```

把170行前面的注释去掉，这里主要是指引用的具体配置文件，后续要管理的服务相关信息在这里进行配置


如果想通过浏览器对supervisor下的进程进行可视化管理的话，就需要把相关的配置打开

```
39 ;[inet_http_server]         ; inet (TCP) server disabled by default
40 ;port=127.0.0.1:9001        ; ip_address:port specifier, *:port for all iface
41 ;username=root              ; default is no username (open server)
42 ;password=123               ; default is no password (open server)
```

把这几行的前面的注释符号 ； 去掉


## 管理

### 1. 命令行模式

```sudo /usr/local/Cellar/supervisor/4.2.2_1/bin/supervisorctl -c /usr/local/etc/supervisord.conf```

通过这条命令进行命令行模式，然后就可以根据具体的命令进行操作了

![cmd](/images/supervisor/cmd_mode.jpg)

常用的命令语句如下：

```
reread ;重新加载配置文件
update ;将配置文件里新增的子进程加入进程组，如果设置了autostart=true则会启动新新增的子进程
status ;查看所有进程状态
status <name> ;查看指定进程状态
start all; 启动所有子进程
start <name>; 启动指定子进程
restart all; 重启所有子进程
restart <name>; 重启指定子进程
stop all; 停止所有子进程
stop <name>; 停止指定子进程
reload ;重启supervisord
add <name>; 添加子进程到进程组
reomve <name>; 从进程组移除子进程，需要先stop。注意：移除后，需要使用reread和update才能重新运行该进程
```


### 2. 浏览器模式

直接在浏览器地址栏输入: http://127.0.0.1:9001/，后面有对应的action指令可以点击操作

![cmd](/images/supervisor/brower_mode.jpg)


## 实例

### 编写demo代码

我写了两个go的demo文件进行验证测试，一个正常执行，一个会触发panic


o-work.go - 正常执行： 每两秒会输出当前的时间


```
package main

import (
	"fmt"
	"time"
)

func main() {
	timer := time.NewTimer(time.Second * 2)
	i := 0
	for {
		select {
		case <-timer.C:
			i++
			fmt.Printf("i:%d time:%s\n", i, time.Now().Format("2006-01-02 15:04:05"))
			timer.Reset(time.Second * 2)
		}
	}
}

```


go-panic-work.go - 非正常执行：每两秒会输出当前的时间，当执行到第10次时，触发panic


```
package main

import (
	"fmt"
	"time"
)

func main() {
	timer := time.NewTimer(time.Second * 2)
	i := 0
	for {
		select {
		case <-timer.C:
			i++
			fmt.Printf("i:%d time:%s\n", i, time.Now().Format("2006-01-02 15:04:05"))
			if i == 10 {
				panic("over")
			} else {
				timer.Reset(time.Second * 2)
			}
		}
	}
}
```

然后分别进行编译 go build go-work.do、go build go-panic-work.do


### 配置部署

因为我们的配置文件都在 /usr/local/etc/supervisor.d/*.ini
首先创建目录 /usr/local/etc/supervisor.d，然后在该目录下创建配置文件 go-work.ini
具体内容如下：

```
  1 [program:go-work] #程序的名字，在supervisor中可以用这个名字来管理该程序
  2 directory=~/code/test  #相当于在该目录下执行程序
  3 command=~/code/test/go-work #启动程序的命令
  4 numprocs=1 #启动的进程数量
  5 autostart=true #supervisor启动的时候是否随着同时启动，默认True
  6 autorestart=true #设置子进程挂掉后自动重启的情况，有三个选项，false,unexpected和true。如果为false的时候，无论什么情况下，都不会被重新启动，如果为unexpected，只有当进程的退出码不在下面的exitcodes里面定义的
  7 startretries=3 #重启程序的次数
  8 user=root #指定运行用户
  9 redirect_stderr=true #是否将程序错误信息重定向的到文件
 10 stdout_logfile=~/log/%(program_name)s.log #将程序输出重定向到该文件
 11 stdout_logfile_maxbytes=300MB #指定日志文件最大字节数，默认为50MB，可以加KB、MB或GB等单位
 12 stdout_logfile_backups=3 #要保存的stdout_logfile备份的数量
```

同样的方法创建 go-panic-work.go 的配置文件

### 加载

通过命令 sudo /usr/local/Cellar/supervisor/4.2.2_1/bin/supervisorctl -c /usr/local/etc/supervisord.conf 进入命令行模式


```
supervisor> reread
go-panic-work: available
go-work: available
``` 

通过 reread 命令 加载刚添加的配置文件，出现available提示时说明成功加载


```
supervisor> update
go-panic-work: added process group
go-work: added process group
```

通过 update 命令 把刚才两个项目添加到工作组中
因为刚才设置了 autostart=true，所以update会启动两个进程


```
supervisor> status
go-panic-work                    RUNNING   pid 10963, uptime 0:00:05
go-work                          RUNNING   pid 10959, uptime 0:00:46
```

最后通过status命令，可以看到刚才两个进程已经被添加进去并正常启动

如果需要移除相关服务，可以通过 stop停止服务，然后remove移出工作组

### 日志

go-work.log

```
i:1 time:2021-12-19 15:46:20
i:2 time:2021-12-19 15:46:22
i:3 time:2021-12-19 15:46:24
i:4 time:2021-12-19 15:46:26
i:5 time:2021-12-19 15:46:28
i:6 time:2021-12-19 15:46:30
i:7 time:2021-12-19 15:46:32
i:8 time:2021-12-19 15:46:34
i:9 time:2021-12-19 15:46:36
i:10 time:2021-12-19 15:46:38
i:11 time:2021-12-19 15:46:40
i:12 time:2021-12-19 15:46:42
i:13 time:2021-12-19 15:46:44
i:14 time:2021-12-19 15:46:46
i:15 time:2021-12-19 15:46:48
...
```

可以看到服务一直在正常执行

go-panic-work.log

```
i:1 time:2021-12-19 16:24:54
i:2 time:2021-12-19 16:24:56
i:3 time:2021-12-19 16:24:58
i:4 time:2021-12-19 16:25:00
i:5 time:2021-12-19 16:25:02
i:6 time:2021-12-19 16:25:04
i:7 time:2021-12-19 16:25:06
i:8 time:2021-12-19 16:25:08
i:9 time:2021-12-19 16:25:10
i:10 time:2021-12-19 16:25:12
panic: over

goroutine 1 [running]:
main.main()
    /Users/jihui/code/test/print_panic.go:17 +0x13d
i:1 time:2021-12-19 16:25:15
i:2 time:2021-12-19 16:25:17
i:3 time:2021-12-19 16:25:19
i:4 time:2021-12-19 16:25:21
i:5 time:2021-12-19 16:25:23
i:6 time:2021-12-19 16:25:25
i:7 time:2021-12-19 16:25:27
i:8 time:2021-12-19 16:25:29
i:9 time:2021-12-19 16:25:31
i:10 time:2021-12-19 16:25:33
panic: over
...
```

可以看到每执行到第10次时候，发生panic，服务挂掉；然后被supervisor重新启动，再次开始执行



## 问题

启动supervisor启动成功后，配置的所有的任务可能无法启动，查看进程日志，出现如下错误：supervisor couldn’t setuid to 0 Can’t drop privilege as nonroot user

这个主要是由于启动supervisor的用户和program配置的用户不是同一个用户导致

### 查看日志
tail -f /usr/local/var/log/supervisord.log

### 解决方案


```
//用root启动supervisord
sudo /usr/local/Cellar/supervisor/4.2.2_1/bin/supervisord -c /usr/local/etc/supervisord.conf
//再用root启动子进程
sudo /usr/local/Cellar/supervisor/4.2.2_1/bin/supervisorctl -c /usr/local/etc/supervisord.conf start go-work
```








