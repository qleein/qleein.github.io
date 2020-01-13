---
layout: post
title:  "Linux下使用tc和netem模拟复杂网络环境"
date:   2019-12-29 20:58:00 +0800
---

netem(Network Emulator)可以用来对网卡发出的数据包进行增加延迟、丢包、重复、乱序等处理，来模拟复杂网络环境。netem的设置依赖tc命令，tc是Linux内核提供的流量控制工具。

# 基本用法

对eth0网卡发出的数据包延迟500毫秒发送

```
$ tc qdisc add dev eth0 root netem delay 500ms


```

这里的qdisc是tc中的一个基本概念，dev eth0表示操作的网卡，root表示出队列的根（egress），与之相对的是ingress(入队列)。netem表示启用netem，delay 500ms表示延迟500毫秒。其他命令类似。

对eth0网卡发出的数据包随机丢包5%

```
$ tc qdisc add dev eth0 root netem loss 5%


```

对eth0网卡发出的数据包产生10%的重复数据包

```
$ tc qdisc add dev eth0 root netem duplicate 10%


```

对eth0网卡发出的数据包，乱序25%（50%相关），延迟50ms

```
$ tc qdisc add dev eth0 root netem delay 50ms reorder 25% 50%


```

# 处理特定IP或端口的数据包

前述的命令会影响指定网络发出的所有数据包，tc命令支持更精细的控制。

如下面的设置，对发往1.1.1.1的数据包随机丢包10%

```
$ tc qdisc add dev eth0 root handle 1: prio
$ tc filter add dev eth0 parent 1:0 protocol ip prio 1 u32 match ip dst 1.1.1.1 flowid 2:1
$ tc qdisc add dev eth0 parent 1:1 handle 2: netem loss 10%

```

上述命令，先在入队列的根节点绑定一个prio qdisc，handle指定队列的id，id为冒号分割的两个数字。prio disc下自带了三个band，id分别是1:0, 1:1, 1:2。然后在band 1：1上增加一个qdisc，类型为netem，loss 10%是netem参数，表示丢包10%。同时，通过tc filter增加规则，将匹配到目的IP为1.1.1.1的数据包交由id为2：1的对象（即netem的 qdisc）。

这里命令中的handle后的id如，1:、1:1等不能重复。详细的解析可以参考tc命令。在filter中配置不同的match规则，可以匹配目的IP、源IP、目的端口、源端口等。

# 处理入数据包

netem只能发出的数据包，无法处理接收的数据包。要处理入数据包，需要借助Linux内核提供的ifb模块(Intermediate Functional Block)。

先加载ifb模块，启动ifb网卡

```
$ modprobe ifb
$ if link set ifb0 up


```

通过tc命令将入流量导入到ifb网卡上。

```
$ tc qdisc add dev eth0 handle ffff: ingress
$ tc filter add dev eth0 parent ffff: u32 match u32 0 0 action mirred egress redirect dev ifb0


```

然后再通过tc命令设置需要的规则即可，只是这里需要操作的是ifb0网卡。 如设置入数据包延迟500ms

```
$ tc qdisc add dev ifb0 root netem delay 500ms


```

设置其他处理方式，或者按照IP/端口处理与前述相同。

# 使用tcconfig配置netem

tc提供的非常强大的功能，同时也非常难以理解使用。通过tcconfig可以方便地配置netem的功能。 tcconfig可以通过pip或pip3安装，详见[官方文档](https://tcconfig.readthedocs.io/en/latest/)，可惜实测centos6通过pip安装会失败。

如对发往1.1.1.1的数据包增加延迟500ms，只需要一个命令即可。tcconfig会自动根据需要调用tc命令进行配置。

```
$ tcset eth0 --delay 500ms --dst-address 1.1.1.1 --direction outgoing 

```

对1.1.1.1的80端口的出入数据包增加随机丢包5%。

```
$ tcset eth0 --loss 5% --dst-address 1.1.1.1 --dst-port 80 --direction outgoing
$ tcset eth0 --loss 5% --src-address 1.1.1.1 --src-port 80 --direction incoming

```

通过tcshow命令可以查看现有的规则

```
$ tcshow eth0

```

通过tcdel命令删除现有规则

```
$ tcdel eth0 --all

```