---
layout: post
title:  "systemd创建定时任务"
date:   2017-03-12 21:28:00 +0800
tags: systemd 定时任务
categories: 日常记录
---

## Systemd or Cron

Cron是类Unix系统里最常见的任务计划程序，而Systemd也开始提供定时器作为Cron的替代品。尽管争议不断，Systemd还是被越来越多的Linux发行版使用，Ubuntu也是如此。因此在需要创建定时任务时我决定向"邪恶势力"低头，基于Systemd来实现。

Systemd配置根据功能划分到不同的单元，如系统服务(.service)属于服务单元，定时器(.timer)则属于定时器单元。每种单元有自己的配置文件格式。

不同于Cron通过crontab一行添加一个定时任务，Systemd要创建定时任务需要创建两个文件，一个定时器单元文件定义定时任务的激活时间，一个服务单元文件定义定时器激活时执行的具体任务。从这个角度想，相当于定时任务激活时通过`systemctl start XXX.service`启动了一个系统服务。不过一个定时任务需要两个文件这种方式还是很麻烦了。

systemd创建定时任务的主要步骤
* 创建服务单元文件
* 创建定时器单元文件
* 启动定时器

## 创建定时器单元文件
定时器单元文件是以timer为后缀的systemd单元文件，定时器单元文件包括三个部分
* Unit
* Timer
* Install

[Unit]部分定义了systemd单元文件通用的一些配置，[Install]部分定义安装systemd单元需要的配置，在systemd执行过程中不会用到，仅在调用`systemctl enable`或者`systemctl disable`设置开机启动时需要。

[Timer]部分定义了何时以及如何激活定时事件。Timers 可以被定义成以下两种类型：
* 单调定时器
* 实时定时器

### 单调定时器
单调定时器 即从一个时间点过一段时间后激活定时任务。所有的单调计时器都遵循如下形式： OnTypeSec=。有以下几种类型

    * OnActiveSec 时间点为定时器启动
    * OnBootSec 时间点为系统启动
    * OnStartupSec 时间点为systemd启动时间
    * OnUnitActiveSec 时间点为上次定时器任务激活时间
    * OnUnitInactiveSec 时间点为上次定时器任务执行完毕的时间

不同的类型可以组合，如下面的配置，在系统启动15分钟后第一次执行，之后每隔1周执行一次。
```
[Timer]
OnBootSec=15min
OnUnitActiveSec=1w 
```

时间间隔的所有定义格式可以通过`man systemd.time`查看。

### 实时定时器

实时定时器 (亦称"挂钟定时器") 通过日历事件激活（类似于 cronjobs ）定时任务。 使用 OnCalender= 来定义实时定时器。

如下面的配置`OnCalendar=Wed, 23:15`表示每周三的23点15分执行。日历事件的详细定义格式可以通过`man systemd.time`查看。

下面是一个定时器单元文件。
```
[Unit]
Description=Run foo weekly and on boot

[Timer]
OnCalendar=Wed, 23:15

[Install]
WantedBy=timers.target
```

## 创建服务单元文件

 服务单元文件也包括三部分，只是没有[Timer]部分，增加了[Service]部分，这部分定于服务执行相关的配置，最常用的是ExecStart，设置服务启动时需要执行的操作。

如下面的服务单元文件，`ExecStart=/usr/local/bin/foo`表示启动时执行foo程序。
```
[Unit]
Description=MyScript

[Service]
ExecStart=/usr/local/bin/foo
```

这里的服务单元只由定时器单元调用，所以不需要[Install]部分。

## 单元文件路径

单元文件要放在Systemd指定的路径才能生效。Systemd单元按照运行模式分两种，system模式和user模式。
system模式只要系统在运行就会生效，而user模式在用户登录状态才会生效。两种模式下单元文件的路径也不同。

Systemd在system模式下加载单元文件的路径，优先级从高到低有
* /etc/systemd/system
* /run/systemd/system
* /lib/systemd/system

Systemd在user模式下加载单元文件的路径，优先级从高到低有
* $XDG_CONFIG_HOME/systemd/user
* $HOME/.config/systemd/user
* /etc/systemd/user
* $XDG_RUNTIME_DIR/systemd/user
* /run/systemd/user  
* $XDG_DATA_HOME/systemd/user
* $HOME/.local/share/systemd/user 
* /usr/lib/systemd/user

创建定时器时，定时器单元文件和服务单元文件必须在同一个目录下，且文件名相同，只是后缀不同(一个为.timer，一个为.service)。如创建名为foo的定时器任务，运行在system模式，可以在/etc/systemd/system/下建立一个foo.timer文件和一个foo.service文件。


## 定时器启动和停止

Systemd的命令行操作是用过systemctl命令实现的。

启动定时器
```
$ systemctl start XXX.timer
```

停止定时器
```
$ systemctl stop XXX.timer
```

设置开机自动启动
```
$ systemctl enable XXX.timer
```

取消开机启动
```
$ systemctl disable XXX.timer
```

## 查看运行状态

查看已启动的所有定时器
```
$ systemctl list-timers
```

查看定时器的运行状态
```
systemctl status XXX.timer
```

定时器运行出错时需要查看日志进行分析，systemd日志为二进制格式，可以通过journal命令查看
```
$ journal -xe
```