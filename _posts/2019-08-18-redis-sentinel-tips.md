---
layout: post
title:  "基于sentinel构建高可用redis集群的注意事项"
date:   2019-08-18 21:15:00 +0800
tags: redis sentinel 
categories: 日常记录
---

部署redis高可用集群时，通常会用到redis官方的sentinel。sentinel监控master状况，master宕机时进行集群master的故障转移。部署时方法网上很多，这里列出了一些需要注意的事项。

### 设置相同的集群密码

为了安全，有时候会对redis和sentinel设置密码，由于集群可能会切换，所以redis集群的密码最好相同，对于将slave-priority设置为0的节点，由于不会成为master，所以密码可以单独设置，不与其他机器相同，只要masterauth设置为master的密码即可。

sentinel在redis5以后的版本支持设置密码，由于sentinel之间需要相互通信，所以整个sentinel集群内密码必须相同。当然不需要与redis节点的密码相同

### redis绑定本地IP

redis配置文件的bind指令可以绑定多个IP, 默认绑定了127.0.0.1。如果sentinel不在本机，需要redis绑定外网IP，这时bind绑定的第一个IP必须可以对外访问。

redis的主从复制需要slave和master通信，而redis对外建立TCP连接时，会使用bind绑定的第一个IP作为本地IP，如配置了`bind 127.0.0.1 1.1.1.1`，slave访问master时，会绑定127.0.0.1，master如果不是本机，会连接失败。

### 配置文件的dir放在其他路径相关指令的前面

dir指令改变后面指令的相对路径，如果配置中有其他依赖相对文件路径的指令，如`pidfile`、`logfile`，要把这些指令放在`dir`指令后面。

### 禁用CONFIG指令导致无法切换master

redis的`CONFIG`指令会修改本地文件，导致安全风险，而在配置文件中加入`rename-command CONFIG ""`，可以禁用`CONFIG`指令。

sentinel检测到master宕机进行故障切换时，会向slave发送一个事务指令（`MULTI`), 包含有`SLAVEOF`和`CONFIG`两个指令，`SLAVEOF`指令指示slave更换master，`CONFIG`指令将slaveof更新到配置文件里。如果禁用了`CONFIG`指令会导致事务失败，相应的slave无法切换master。

### sentinel机器无外网IP需通过announce-ip指定

redis和sentinel都支持`announce-ip`和`announce-port`声明ip和端口，如果对外提供服务的IP与对外访问的出口不一致时需要指定这个配置，尤其现在很多云主机都是只有内网IP，通过slb对外访问的。

sentinel之间的相互发现是靠master来实现的，sentienl向master的`__sentinel__:hello"`的channel广播自己的IP地址，其他sentinel根据这个消息发现其他的sentinel。如果sentienl的机器是内网的，而sentinel之间需要外网访问，就需要在配置文件里，将`announce-ip`配置为外网可以访问的IP。而redis之间，地址通过TCP连接的对端地址获取，不存在这个问题。

### 向运行中sentinel增加带有密码的redis集群

sentinel通过读取配置文件获取master信息，同时运行中的sentinel也可以通过指令配置。

在通过`sentinel monitor`可以增加监控的集群时，如果集群带有密码，通过`sentinel set 集群名 auth-pass password`来设置。设置后需要通过`sentinel reset 集群名`，让sentinel重新与集群master建立TCP连接才能真正生效。

### 不要对已加入集群的redis执行手动执行slaveof

sentinel会监控集群中节点的状态，如果对redis集群内的机器手动执行slaveof执行，sentinel发现后会重新发送slaveof指令，保证集群的正常运行。

### 从集群内移除节点

如果节点是slave，先将节点宕机，然后向所有监控这个集群的sentinel，发送`sentinel reset 集群名`重置集群节点信息，将节点从集群彻底移除，保证所有的sentinel都执行了这个指令，否则sentinel会重新发送`slaveof`指令将节点加入集群。

如果节点是master，需要切换master，后照slave的情况进行。

移除sentinel节点，与移除slave方法相同。

### redis集群多层级

redis主从复制支持多层级，如果使用了sentinel，集群的管理可能出现混乱。

如node1有两个从节点node2和node3，node2有一个从节点node4。如果node1宕机，sentinel将集群的master切换到了node2,那么node3和node4就变成了同一个层级的，这时sentinel该如果处理？不过并没有实际进行验证，不确定的情况下最好避免这种情形。

### sentinel故障切换时如何选择slave

sentinel发现master宕机时，改为每一秒向所有的slave发送`INFO`指令，以及时获取slave状态。之后需要选择一个slave提升为集群master。

在选择slave时，将连接正常，`且slave-priority`不为0的slave组成一个数组，按照节点的`slave-priority`从小到大排序，对于`slave-priority`值相同的节点，依照节点的`slave_repl_offset`值从大到小排序（`slave_repl_offset`为主从复制的偏移，值越大说明节点的数据越接近master）。排序完成后，选择数组中最前面的节点，提升为集群master。

### redis集群数据不满足强一致性

redis主从复制并没有提供强一致性，所以从slave读到的数据与master可能有延迟。且master故障切换时数据可能会丢失。

senetinel并不关注redis主从复制的状态与延迟，选择sentinel时也不会保证将拥有最新数据的slave提升为master。

### 新部署的slave在同步完成前没有数据

新部署的slave，同master连接后，master生成一个rdb文件传输给slave，slave加载其中的数据。在这个过程中，slave是可以对外访问的，导致这个期间请求获取的结果都是不存在的。只有在slave收到rdb文件，加载的过程中，redis无法对外服务。

