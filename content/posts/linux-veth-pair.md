---
title: "Linux veth pair 详解"
categories: [ "Linux网络" ]
tags: [ "linux","network","veth pair" ]
draft: false
slug: "linux-veth-pair"
date: "2020-11-09 22:45:00"
---

# Linux veth pair 详解


veth pair是成对出现的一种虚拟网络设备接口，一端连着网络协议栈，一端彼此相连。如下图所示：


![virtual-device-veth-1](https://oss.typesafe.cn/virtual-device-veth-1.png)


由于它的这个特性，常常被用于构建虚拟网络拓扑。例如连接两个不同的网络命名空间(netns)，连接docker容器，连接网桥(Bridge)等，其中一个很常见的案例就是OpenStack Neutron底层用它来构建非常复杂的网络拓扑。


## 如何使用？


创建一对veth


```
ip link add <veth name> type veth peer name <peer name>
```


## 实验
我们改造上一节完成的netns实验，使用veth pair将两个的隔离netns连接起来。如下图所示：


![https://oss.typesafe.cn/vethpair.png](https://oss.typesafe.cn/vethpair.png)


我们首先创建一对veth设备，将veth设备分别移动到两个netns中并启动。


```bash
# 创建一对veth
ip link add veth0 type veth peer name veth1
# 将veth移动到netns中
ip link set veth0 netns ns0
ip link set veth1 netns ns1
# 启动
ip netns exec ns0 ip link set veth0 up
ip netns exec ns1 ip link set veth1 up
```


接下来我们测试一下。


使用`ip netns exec ns0 ping 10.0.0.2`在命名空间ns0中测试与tap1的网络连通性。

```
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
From 10.0.0.1 icmp_seq=1 Destination Host Unreachable
From 10.0.0.1 icmp_seq=2 Destination Host Unreachable
From 10.0.0.1 icmp_seq=3 Destination Host Unreachable
^C
--- 10.0.0.2 ping statistics ---
5 packets transmitted, 0 received, +3 errors, 100% packet loss, time 77ms
pipe 4
```

使用`ip netns exec ns1 ping 10.0.0.1`在命名空间ns1中测试与tap0的网络连通性。

```
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
From 10.0.0.2 icmp_seq=1 Destination Host Unreachable
From 10.0.0.2 icmp_seq=2 Destination Host Unreachable
From 10.0.0.2 icmp_seq=3 Destination Host Unreachable
^C
--- 10.0.0.1 ping statistics ---
4 packets transmitted, 0 received, +3 errors, 100% packet loss, time 108ms
pipe 4
```

什么情况？为什么网络还是不通呢？答案就是路由配置有问题。

使用`ip netns exec ns0 route -n`查看ns0的路由表。

```
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 tap0
```

使用`ip netns exec ns1 route -n`查看ns1的路由表。
```
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 tap1
```

原来访问10.0.0.0/24的流量都从tap设备发出去了，又因为tap设备没有和其他设备相连，发出去的数据报文不会被处理，因此还是访问不到目标IP，我们来修改一下路由，让访问10.0.0.0/24的流量从veth设备发出。

```
#修改路由出口为veth
ip netns exec ns0 ip route change 10.0.0.0/24 via 0.0.0.0 dev veth0
ip netns exec ns1 ip route change 10.0.0.0/24 via 0.0.0.0 dev veth1
```

我们再来看一下路由

使用`ip netns exec ns0 route -n`查看ns0的路由表。

```
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 veth0
```

使用`ip netns exec ns1 route -n`查看ns1的路由表。
```
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 veth1
```

最后我们再来测试一下。

使用`ip netns exec ns0 ping 10.0.0.2`在命名空间ns0中测试与tap1的网络连通性。

```bash
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.031 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.035 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.037 ms
64 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=0.043 ms
^C
--- 10.0.0.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 103ms
rtt min/avg/max/mdev = 0.031/0.036/0.043/0.007 ms
```

使用`ip netns exec ns1 ping 10.0.0.1`在命名空间ns1中测试与tap0的网络连通性。


```bash
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.027 ms
64 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=0.047 ms
64 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=0.051 ms
64 bytes from 10.0.0.1: icmp_seq=4 ttl=64 time=0.042 ms
^C
--- 10.0.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 66ms
rtt min/avg/max/mdev = 0.027/0.041/0.051/0.012 ms
```


可以看到我们使用veth pair将两个隔离的netns成功的连接到了一起。


但是这样的网络拓扑存在一个弊端，随着网络设备的增多，网络连线的复杂度将成倍增长。
如果连接三个netns时，网络连线就成了下图的样子

![veth1](https://oss.typesafe.cn/veth-pair1.png)

而如果连接四个netns时，网络连线就成了下图的样子

![veth2](https://oss.typesafe.cn/veth-pair2.png)

如果有五台设备。。。


有没有什么技术可以解决这个问题呢？答案是有的，Linux Bridge（网桥）。下一节我们将使用网桥来将多个隔离的netns连接起来，这样网络连线就非常清爽了。