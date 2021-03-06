---
title: "Linux 环回网络接口"
categories: [ "Linux网络" ]
tags: [ "linux","network","loopback" ]
draft: false
slug: "linux-loopback"
date: "2021-01-28 22:59:00"
---

在开发或者调试时，我们经常需要和本地的服务器进行通信，例如启动`nginx`之后，在浏览器输入`lcoalhost`或者`127.0.0.1`就可以访问到本机上面的`http`服务。

## Linux是如何访问本机IP的？

大多数操作系统都在网络层实现了环回能力，通常是使用一个虚拟的环回网络接口来实现。这个虚拟的环回网络接口看着像是一个真实的网卡，实际上是操作系统用软件模拟的，它可以通过`TCP/IP`与同一台主机上的其他服务进行通信，以`127`开头的`IPv4`地址就是为它保留的，主流`Linux`操作系统为环回网卡分配的地址都是`127.0.0.1`，主机名是`localhost`。

环回网络接口之所以被称之为环回网络接口，是因为从本机发送到本机任意一个IP的数据报文都会在网络层交给环回网络接口，不再下发到数据链路层进行处理，环回网络接口直接发送回网络层，最终交由应用层软件程序进行处理。这种方式对于性能测试非常有用，因为省去了硬件的开销，可以直接测试协议栈软件所需要的时间。

那环回网络接口是如何判断目的IP是否为本机地址的呢？

答案就是网络层在进行路由转发的时候会先查本地的路由表，发现是本机IP后交给环回网络接口。查看本地路由表的命令如下：

``` shell
ip route show table local
```

输出内容如下：

``` shell
broadcast 10.141.128.0 dev eth0 proto kernel scope link src 10.141.155.131 
local 10.141.155.131 dev eth0 proto kernel scope host src 10.141.155.131 
broadcast 10.141.191.255 dev eth0 proto kernel scope link src 10.141.155.131 
broadcast 127.0.0.0 dev lo proto kernel scope link src 127.0.0.1 
local 127.0.0.0/8 dev lo proto kernel scope host src 127.0.0.1 
local 127.0.0.1 dev lo proto kernel scope host src 127.0.0.1
```

其中`local`开头的便是本地IP，`dev`后面是网卡名称。

查完了本地路由表之后会再查主路由表，也就是我们经常操作的路由表。

``` shell
ip route show table main
```

输出内容如下

``` shell
default via 10.141.128.1 dev eth0 proto static metric 100 
10.141.128.0/18 dev eth0 proto kernel scope link src 10.141.155.131 metric 100
```

## 环回网络接口

现在我们再来看下环回网络接口

``` shell
ifconfig lo
```

输出

``` shell
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 1554227  bytes 123327716 (117.6 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1554227  bytes 123327716 (117.6 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

可以看到本地环回接口的`IPv4`地址是`127.0.0.1`，子网掩码是`255.0.0.0`，对应A类网络号`127`，有趣的是当我们访问 `127.0.0.1-127.255.255.254`之间的任意一个地址都会访问到本机。

`IPv6`地址`::1`，前缀是`128`位，表示只有一个地址。

环回网络接口的当前`MTU`是`64KB`，不过最高可以设置到`2GB`，真是恐怖如斯。

下面几条`RX`,`TX`开头的分别代表收发到的数据报文个数和大小以及错包、丢包、溢出次数和无效帧。

## FAQ

#### 虚拟网卡的IP属于本机IP吗？

属于，因为与宿主机器共用同一个网络协议栈。

#### 宿主机器上创建netns，netns内部的IP属于本机IP吗？

不属于，因为netns拥有独立的网络协议栈，在netns内部也可以看到它本身的环回网络接口。