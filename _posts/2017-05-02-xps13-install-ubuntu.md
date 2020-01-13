---
layout: post
title:  "xps 13安装ubuntu 16.04"
date:   2017-05-02 22:27:00 +0800
categories: 日常记录
---

入手戴尔的xps13，预装的Win10操作系统。考虑到这款机型有预装ubuntu 16.04的开发者版本，只是没有在国内发售，遂询问戴尔客服，是否可以提供系统镜像，或者相应的驱动程序。对此戴尔方面表示他们官方没有提供任何驱动程序，系统默认是有适配的，直接安装ubuntu即可......

网上搜了下，貌似有人尝试过。如这里 [在Dell XPS 13安装WIN10和ubuntu双系统](http://blog.csdn.net/james_wu_shanghai/article/details/50976347)，于是开始动手。

# 更改硬盘模式

xps 13 默认采用raid模式，这里改成ACHI模式。

1. 按电源键启动，出现dell图标时按F2 进入bios设置
2. 左边列表框中选择`Settings -> System Configuration -> SATA Operation`，在右侧的`SATA Operation`中选择AHCI`
3. 点击`Apply`确认

# Secure Boot

Ubuntu 16.04已支持Secure Boot，所以不需要关闭Secure Boot。

# 安装Ubuntu 16.04

用准备好的启动盘安装系统。没有保留Win10系统，所以直接格式化然后重新分区了。

安装时有提示现在硬盘有安装操作系统，重新分区可能导致之前的系统无法启动，可以选择是否继续，这里点击continue居然卡住了，无奈按电源键强制关机，第二次就没问题了正常安装。

# 网卡驱动

xps 13没有RJ-45网线接口，手头也没有转接头，只能通过无线网卡连接网络了。刚安装好的Ubuntu系统无法用无线，应该是驱动的问题。搜索了下，发现安装一个[固件](https://launchpad.net/ubuntu/xenial/amd64/linux-firmware/1.157.5)就可以解决了。

1. 下载安装包 [http://launchpadlibrarian.net/292156147/linux-firmware_1.157.5_all.deb](http://launchpadlibrarian.net/292156147/linux-firmware_1.157.5_all.deb)
2. 用u盘拷贝到xps 13上
3. sudo dpkg -i linux-firmware_1.157.5_all.deb
4. 重启机器

# 更新软件包

新安装的Ubuntu， 一旦合上电脑休眠后就会假死，只能通过按电源键强制关机。貌似新的版本修复了这个问题，只要更新所有的软件包就正常了。

在终端执行`sudo apt update && sudo apt upgrade`即可

至此安装完成，已经可以正常使用了。

# 触控板设置

Ubuntu默认的触控板有下面几种操作
单指单击不说了
双指上下滑上下滚动
双指左右滑左右滚动
双指单击相当于鼠标右键
三指双击（单击无效果）切换窗口
四指单击相当于super

如果想要更多的手势操作，就需要xSwipe之类的程序了

# 设置开机启动

在终端输入`gnome-session-properties`可以启动开机启动项的设置

# 设置Socks代理

图形界面的客户端[安装方法](https://github.com/shadowsocks/shadowsocks-qt5/wiki/Installation)
命令行客户端通过`pip install shadowsocks`安装

# chrome使用Socks代理

chrome插件Proxy SwitchyOmega可以管理socks代理设置。然而chrome安装插件需要通过代理才行。这里可以通过命令行启动chrome。

在终端中输入`google-chrome --proxy-server="socks5://(socks5的监听ip):(监听端口)"`启动chrome。安装插件后退出，之后就可以通过Proxy SwitchyOmega控制代理设置了。




