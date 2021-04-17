---
title: "Linux Bridge 详解"
categories: [ "Linux网络" ]
tags: [ "linux","network","bridge" ]
draft: false
slug: "linux-bridge"
date: "2020-11-13 21:47:00"
---

# Linux Bridge 详解

Linux Bridge（网桥）是用纯软件实现的虚拟交换机，有着和物理交换机相同的功能，例如二层交换，MAC地址学习等。因此我们可以把tun/tap，veth pair等设备绑定到网桥上，就像是把设备连接到物理交换机上一样。此外它和veth pair、tun/tap一样，也是一种虚拟网络设备，具有虚拟设备的所有特性，例如配置IP，MAC地址等。

Linux Bridge通常是搭配KVM、docker等虚拟化技术一起使用的，用于构建虚拟网络，因为此教程不涉及虚拟化技术，我们就使用前面学习过的netns来模拟虚拟设备。

## 如何使用Linux Bridge？

操作网桥有多种方式，在这里我们介绍一下通过**bridge-utils**来操作，由于它不是Linux系统自带的工具，因此需要我们手动来安装它。

```bash
# centos
yum install -y bridge-utils
# ubuntu
apt-get install -y bridge-utils
```

使用`brctl help`查看使用帮助

```bash
never heard of command [help]
Usage: brctl [commands]
commands:
	addbr     	<bridge>		add bridge
	delbr     	<bridge>		delete bridge
	addif     	<bridge> <device>	add interface to bridge
	delif     	<bridge> <device>	delete interface from bridge
	hairpin   	<bridge> <port> {on|off}	turn hairpin on/off
	setageing 	<bridge> <time>		set ageing time
	setbridgeprio	<bridge> <prio>		set bridge priority
	setfd     	<bridge> <time>		set bridge forward delay
	sethello  	<bridge> <time>		set hello time
	setmaxage 	<bridge> <time>		set max message age
	setpathcost	<bridge> <port> <cost>	set path cost
	setportprio	<bridge> <port> <prio>	set port priority
	show      	[ <bridge> ]		show a list of bridges
	showmacs  	<bridge>		show a list of mac addrs
	showstp   	<bridge>		show bridge stp info
	stp       	<bridge> {on|off}	turn stp on/off
```

常用命令如

新建一个网桥：

```bash
brctl addbr <bridge>
```

添加一个设备（例如`eth0`）到网桥：

```bash
brctl addif <bridge> eth0
```

显示当前存在的网桥及其所连接的网络端口：

```bash
brctl show
```

启动网桥：

```bash
ip link set <bridge> up
```

删除网桥，需要先关闭它：

```bash
ip link set <bridge> down
brctl delbr <bridge>
```

或者使用`ip link del` 命令直接删除网桥

```bash
ip link del <bridge>
```

> 增加Linux Bridge时会自动增加一个同名虚拟网卡在宿主机器上，因此我们可以通过`ip link`命令操作这个虚拟网卡，实际上也就是操作网桥，并且只有当这个虚拟网卡状态处于**up**的时候，网桥才会转发数据。

## 实验

在上一节《Linux veth pair详解》我们使用veth pair将两个隔离的netns连接在了一起，在现实世界里等同于用一根网线把两台电脑连接在了一起，但是在现实世界里往往很少会有人这样使用。因为一台设备不仅仅只需要和另一台设备通信，它需要和很多很多的网络设备进行通信，如果还使用这样的方式，需要十分复杂的网络接线，并且现实世界中的普通网络设备也没有那么多网络接口。

那么，想要让某一台设备和很多网络设备都可以通信需要如何去做呢？在我们的日常生活中，除了手机和电脑，最常见的网络设备就是路由器了，我们的手机连上WI-FI，电脑插到路由器上，等待从路由器的DHCP服务器上获取到IP，他们就可以相互通信了，这便是路由器的二层交换功能在工作。Linux Bridge最主要的功能就是二层交换，是对现实世界二层交换机的模拟，我们稍微改动一下网络拓扑，如下图：

![bridge](https://oss.typesafe.cn/bridge0.png)

我们建立了一个网桥，三个netns，三对veth pair，分别一端在netns中，另一端连接在网桥上，为了简化拓扑，我去除了netns中的tap设备，将IP直接配置在veth上。

> veth设备不仅仅可以可以充当“网线”，同时它也可以当作虚拟网卡来使用。

```bash
# 添加网桥
brctl addbr br0
# 启动网桥
ip link set br0 up

# 新增三个netns
ip netns add ns0
ip netns add ns1
ip netns add ns2

# 新增两对veth
ip link add veth0-ns type veth peer name veth0-br
ip link add veth1-ns type veth peer name veth1-br
ip link add veth2-ns type veth peer name veth2-br

# 将veth的一端移动到netns中
ip link set veth0-ns netns ns0
ip link set veth1-ns netns ns1
ip link set veth2-ns netns ns2

# 将netns中的本地环回和veth启动并配置IP
ip netns exec ns0 ip link set lo up
ip netns exec ns0 ip link set veth0-ns up
ip netns exec ns0 ip addr add 10.0.0.1/24 dev veth0-ns

ip netns exec ns1 ip link set lo up
ip netns exec ns1 ip link set veth1-ns up
ip netns exec ns1 ip addr add 10.0.0.2/24 dev veth1-ns

ip netns exec ns2 ip link set lo up
ip netns exec ns2 ip link set veth2-ns up
ip netns exec ns2 ip addr add 10.0.0.3/24 dev veth2-ns

# 将veth的另一端启动并挂载到网桥上
ip link set veth0-br up
ip link set veth1-br up
ip link set veth2-br up
brctl addif br0 veth0-br
brctl addif br0 veth1-br
brctl addif br0 veth2-br
```

测试网络连通性

使用`ip netns exec ns0 ping 10.0.0.2`在命名空间ns0中测试与ns1的10.0.0.2的网络连通性

```bash
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.032 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.058 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.052 ms
64 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=0.044 ms
^C
--- 10.0.0.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 54ms
rtt min/avg/max/mdev = 0.032/0.046/0.058/0.011 ms
```

使用`ip netns exec ns0 ping 10.0.0.3`在命名空间ns0中测试与ns2的10.0.0.3的网络连通性

```bash
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
64 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=0.054 ms
64 bytes from 10.0.0.3: icmp_seq=2 ttl=64 time=0.045 ms
64 bytes from 10.0.0.3: icmp_seq=3 ttl=64 time=0.058 ms
64 bytes from 10.0.0.3: icmp_seq=4 ttl=64 time=0.064 ms
^C
--- 10.0.0.3 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 81ms
rtt min/avg/max/mdev = 0.045/0.055/0.064/0.008 ms
```

使用`ip netns exec ns1 ping 10.0.0.1`在命名空间ns1中测试与ns0的10.0.0.1的网络连通性

```bash
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.031 ms
64 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=0.046 ms
64 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=0.038 ms
64 bytes from 10.0.0.1: icmp_seq=4 ttl=64 time=0.041 ms
^C
--- 10.0.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 81ms
rtt min/avg/max/mdev = 0.031/0.039/0.046/0.005 ms
```

使用`ip netns exec ns1 ping 10.0.0.3`在命名空间ns1中测试与ns2的10.0.0.3的网络连通性

```bash
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
64 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=0.060 ms
64 bytes from 10.0.0.3: icmp_seq=2 ttl=64 time=0.059 ms
64 bytes from 10.0.0.3: icmp_seq=3 ttl=64 time=0.044 ms
64 bytes from 10.0.0.3: icmp_seq=4 ttl=64 time=0.065 ms
^C
--- 10.0.0.3 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 65ms
rtt min/avg/max/mdev = 0.044/0.057/0.065/0.007 ms
```

使用`ip netns exec ns2 ping 10.0.0.1`在命名空间ns2中测试与ns0的10.0.0.1的网络连通性

```bash
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.032 ms
64 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=0.056 ms
64 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=0.043 ms
64 bytes from 10.0.0.1: icmp_seq=4 ttl=64 time=0.060 ms
^C
--- 10.0.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 69ms
rtt min/avg/max/mdev = 0.032/0.047/0.060/0.013 ms
```

使用`ip netns exec ns2 ping 10.0.0.2`在命名空间ns2中测试与ns1的10.0.0.2的网络连通性

```bash
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.030 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.055 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.044 ms
64 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=0.042 ms
^C
--- 10.0.0.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 114ms
rtt min/avg/max/mdev = 0.030/0.042/0.055/0.011 ms
```

可以看到我们通过网桥的方式把三个隔离的netns连接在了一起，通过这种方式，我们还可以很方便的添加第四个netns，第五个netns...在这里我们就不展开了，感兴趣的同学可以尝试一下。

接下来我们来讲解一下docker的几种网络模式。
