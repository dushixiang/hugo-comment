---
title: "Linux Network Namespace (netns) 详解"
categories: [ "Linux网络" ]
tags: [ "linux","network","netns" ]
draft: false
slug: "linux-netns"
date: "2020-11-08 23:12:00"
---


Network Namespace （以下简称netns）是Linux内核提供的一项实现网络隔离的功能，它能隔离多个不同的网络空间，并且各自拥有独立的网络协议栈，这其中便包括了网络接口（网卡），路由表，iptables规则等。例如大名鼎鼎的docker便是基于netns实现的网络隔离，今天我们就来手动实验一下netns的隔离特性。

### 使用方式

使用`ip netns help`查看使用帮助

```bash
Usage: ip netns list
       ip netns add NAME
       ip netns set NAME NETNSID
       ip [-all] netns delete [NAME]
       ip netns identify [PID]
       ip netns pids NAME
       ip [-all] netns exec [NAME] cmd ...
       ip netns monitor
       ip netns list-id
```

### 开始实验

我们将要构建如下图的网络

![https://oss.typesafe.cn/netns.png](https://oss.typesafe.cn/netns.png?t=2)

首先我们添加两个tap设备并配置上IP信息，然后添加两个netns，最后将tap设备移动到netns中

```bash
# 添加并启动虚拟网卡tap设备
ip tuntap add dev tap0 mode tap 
ip tuntap add dev tap1 mode tap 
ip link set tap0 up
ip link set tap1 up
# 配置IP
ip addr add 10.0.0.1/24 dev tap0
ip addr add 10.0.0.2/24 dev tap1
# 添加netns
ip netns add ns0
ip netns add ns1
# 将虚拟网卡tap0，tap1分别移动到ns0和ns1中
ip link set tap0 netns ns0
ip link set tap1 netns ns1
```

在宿主机器上使用`ping 10.0.0.1`测试与tap0的网络连通性

```bash
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
^C
--- 10.0.0.1 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 58ms
```

在宿主机器上使用`ping 10.0.0.2`测试与tap1的网络连通性

``` shell
ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
^C
--- 10.0.0.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 36ms
```

> 由于长时间未收到ICMP的回复报文，我使用Ctrl+C退出了。

使用`ip netns exec ns0 ping 10.0.0.2`在命名空间ns0中测试与tap1的网络连通性

``` shell
connect: 网络不可达
```

使用`ip netns exec ns1 ping 10.0.0.1`在命名空间ns1中测试与tap0的网络连通性

``` shell
connect: 网络不可达
```

> 在netns中执行命令有两种方式，一种是先在宿主机器上执行`ip netns exec <netns name> bash`进入netns，然后就可以像是在本机一样执行命令了。另一种是每次在宿主机器上使用完整的命令，为了明显区分，我们这里都使用完整的命令，例如`ip netns exec ns0 ping 10.0.0.2`的含义为在命名空间**ns0**中执行`ping 10.0.0.2`命令

可以看到在宿主机器上访问netns是丢包，而在netns中互相访问是**网络不可达**了，这是为什么呢？让我们来检查一下netns吧。

使用`ip netns exec ns0 ip a`在ns0中查看网卡

```bash
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
16: tap0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 42:ad:98:a2:cc:81 brd ff:ff:ff:ff:ff:ff
```

使用`ip netns exec ns1 ip a`在ns1中查看网卡

```bash
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
17: tap1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 12:06:1d:06:41:57 brd ff:ff:ff:ff:ff:ff
```

 **可以看到不仅本地环回lo和tap设备的状态都是DOWN，甚至就连tap设备的IP信息也没有了，这是因为在不同的网络命名空间中移动虚拟网络接口时会重置虚拟网络接口的状态。**

我们将ns0和ns1中的相关设备都重新启动并配置上IP

```bash
ip netns exec ns0 ip link set lo up
ip netns exec ns0 ip link set tap0 up
ip netns exec ns0 ip addr add 10.0.0.1/24 dev tap0

ip netns exec ns1 ip link set lo up
ip netns exec ns1 ip link set tap1 up
ip netns exec ns1 ip addr add 10.0.0.2/24 dev tap1
```

首先我们测试一下netns中本地网络是否正常

使用`ip netns exec ns0 ping 10.0.0.1`在命名空间ns0中测试本地网卡是否启动

``` shell
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.036 ms
64 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=0.033 ms
64 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=0.084 ms
64 bytes from 10.0.0.1: icmp_seq=4 ttl=64 time=0.044 ms
^C
--- 10.0.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 65ms
rtt min/avg/max/mdev = 0.033/0.049/0.084/0.021 ms
```

使用`ip netns exec ns1 ping 10.0.0.2`在命名空间ns1中测试本地网卡是否启动

``` shell
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.033 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.034 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.065 ms
64 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=0.035 ms
^C
--- 10.0.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 65ms
rtt min/avg/max/mdev = 0.033/0.049/0.084/0.021 ms
```
可以看出本地网络没有问题，然后我们再来测试一下两个netns之间的网络连通性

使用`ip netns exec ns0 ping 10.0.0.2`在命名空间ns0中测试与tap1的网络连通性

``` shell
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
^C
--- 10.0.0.2 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 84ms
```

使用`ip netns exec ns1 ping 10.0.0.1`在命名空间ns1中测试与tap0的网络连通性

``` shell
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
^C
--- 10.0.0.1 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 30ms
```

可以看出没有任何ICMP回复包，netns确实把在同一台主机上的两张虚拟网卡隔离起来了。在这里我们只是简单的使用`ping`命令来测试网络的连通性，实际上可以做到更多，例如修改某一个netns的路由表或者防火墙规则，完全不会影响到其他的netns，当然也不会影响到宿主机器，在这里由于篇幅原因就不再展开实验了，感兴趣的同学可以实验一下。下一节我们将学习另一个网络设备veth pair，使用它来把两个netns连接起来，让两个隔离的​netns之间可以互相通信。