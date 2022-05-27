---
title: "使用多个WAN IPv4地址"
date: 2019-01-03T21:30:51+08:00
tags: ["网络", "Linux", "配置"]
categories: ["网络"]
draft: false
---
本文介绍一种网络使用情景：在一台PC上使用多个外网地址，并且可以允许应用程序指定外网地址。

**现有设备如下：**

1. 多个电信光猫，每个光猫都能获取独立的公网IP地址。
1. 普通交换机一台，有无VLAN功能都行。
1. 单网卡PC一台。操作系统选用ArchLinux。也可以用Windows做Host，虚拟机中安装ArchLinux。
<!--more-->

**分析**

1. 有些路由带有多拨功能，如openwrt，因此可以使用多个外网地址。然而多拨后的路由通常只支持负载平衡或者带宽最大化，内网的应用程序无法指定外网地址。
1. PC只有一块网卡，因此所有光猫的LAN口都直接连接到交换机，再由交换机连接到PC网卡。
1. 光猫默认开启了DHCP，但是所有光猫的LAN口都连接在一起，DHCP无法正常工作，所以需要用静态IP。
1. 为了方便管理，每个光猫都设置一个独立的网段。在这里将光猫的地址设置为192.168.x.1/24，然后给PC设置多个IP地址192.168.x.2/24。
1. 平常使用的路由表多是基于目标地址的。但是对于外网地址，路由表是用默认路由作为数据包的下一跳。那么在多个默认路由的情况下，只能使用基于源地址的路由表。
1. Windows的路由表不支持基于源地址的路由，因此选择Linux操作系统。本人尝试过Ubuntu Server 16.04但未成功，具体原因不清楚。最后选择了ArchLinux。
1. 完成源地址路由后，用户就可以通过BSD Socket的bind()绑定源地址，这样不同的源地址就对应不同的外网地址。但对于不支持指定源地址的程序仍然无法实现，例如curl。
1. 对于类似curl这种支持代理的程序，我们可以设置一个支持指定源地址的代理程序，通过不同的代理端口来区分不同的外网地址。

**网络拓扑结构如下图：**
![Multi WAN](/img/Multi-WAN.svg)

**PC上的配置**

1. ArchLinux的安装过程参见：[Installation guide](https://wiki.archlinux.org/index.php/Installation_guide)。Linux新手可以选择安装基于ArchLinux的[Manjaro](https://manjaro.org/)。
1. 基于[Netctl](https://wiki.archlinux.org/index.php/Netctl)设置静态IP。

    配置文件：`/etc/netctl/eth0`
    ```bash
    Description='A basic static ethernet connection'
    Interface=eth0
    Connection=ethernet
    IP=static
    Address=('192.168.0.2/24' '192.168.1.2/24' '192.168.2.2/24')
    Gateway='192.168.0.1'
    DNS=('192.168.0.1')
    ```
    应用设置： `bash sudo netctl restart eth0`
1. 创建shell脚本来设置源路由:

    ```bash
    #!/bin/sh
    for ((i=0; i<=2; i++))
    do
        tb=1$i
        ip route flush $tb
        ip route add table $tb to 192.168.$i.0/24 dev eth0
        ip route add table $tb to default  via 192.168.$i.1 dev eth0
        ip rule add from 192.168.$i.0/24 table $tb priority $tb
    done
    ```
1. 用Apache HTTP Server作为代理，配置如下：
    ```apacheConf
    Listen 10000
    Listen 10001
    Listen 10002
    Listen 10003
    <VirtualHost *:10000>
        ProxyRequests On
        ProxyVia Off
        ProxySourceAddress 192.168.0.2
    </VirtualHost>
    <VirtualHost *:10001>
        ProxyRequests On
        ProxyVia Off
        ProxySourceAddress 192.168.1.2
    </VirtualHost>
    <VirtualHost *:10002>
        ProxyRequests On
        ProxyVia Off
        ProxySourceAddress 192.168.2.2
    </VirtualHost>
    ```

**验证**
```shell
$ curl -x localhost:10000 https://ip.cn
当前 IP: 172.0.0.1
$ curl -x localhost:10001 https://ip.cn
当前 IP: 172.0.0.2
$ curl -x localhost:10002 https://ip.cn
当前 IP: 172.0.0.3
```
