---
title: "Open vSwitch 入门实践（2）使用OVS构建隔离网络"
categories: [ "Open vSwitch" ]
tags: [ "sdn","ovs" ]
draft: false
slug: "ovs-learn-2"
date: "2020-11-26 22:55:00"
---

# 前言

在前面我们已经使用Linux Bridge完成了多台网络设备的通信，但是它对于网络隔离的支持不是很好，长期以来，在Linux平台上缺少一个功能完备的虚拟交换机，直到OVS的出现。

# 实验

接下来我们来尝试完成两个实验，单机无隔离网络、单机隔离网络。

## 实验一：单机无隔离网络

使用ovs构建无隔离网络非常简单，只需要添加一个网桥，然后在这个网桥上再增加几个内部端口，最后把端口移动到netns中即可。

```bash
# 添加网桥
ovs-vsctl add-br br-int
# 添加三个内部端口
ovs-vsctl add-port br-int vnet0 -- set Interface vnet0 type=internal
ovs-vsctl add-port br-int vnet1 -- set Interface vnet1 type=internal
ovs-vsctl add-port br-int vnet2 -- set Interface vnet2 type=internal
# 添加三个netns
ip netns add ns0
ip netns add ns1
ip netns add ns2
# 将内部端口分别移动到netns中
ip link set vnet0 netns ns0
ip link set vnet1 netns ns1
ip link set vnet2 netns ns2

# 启动端口并配置IP
ip netns exec ns0 ip link set lo up
ip netns exec ns0 ip link set vnet0 up
ip netns exec ns0 ip addr add 10.0.0.1/24 dev vnet0

ip netns exec ns1 ip link set lo up
ip netns exec ns1 ip link set vnet1 up
ip netns exec ns1 ip addr add 10.0.0.2/24 dev vnet1

ip netns exec ns2 ip link set lo up
ip netns exec ns2 ip link set vnet2 up
ip netns exec ns2 ip addr add 10.0.0.3/24 dev vnet2
```

测试

测试`ns0`和`ns1`能否通信`ip netns exec ns0 ping 10.0.0.2`

```bash
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=1.05 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.059 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.056 ms
64 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=0.053 ms
^C
--- 10.0.0.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3000ms
rtt min/avg/max/mdev = 0.053/0.304/1.051/0.431 ms
```

测试`ns0`和`ns2`能否通信`ip netns exec ns0 ping 10.0.0.3`

```
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
64 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=1.17 ms
64 bytes from 10.0.0.3: icmp_seq=2 ttl=64 time=0.067 ms
64 bytes from 10.0.0.3: icmp_seq=3 ttl=64 time=0.058 ms
64 bytes from 10.0.0.3: icmp_seq=4 ttl=64 time=0.064 ms
^C
--- 10.0.0.3 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3001ms
rtt min/avg/max/mdev = 0.058/0.341/1.177/0.482 ms
```

根据测试结果可以看到，三台设备都是可以互相访问的，这样我们就成功搭建了一个无隔离的二层互通网络。

## 实验二： 单机隔离网络

使用ovs构建隔离网络也很简单，只需要给相应的端口设置上VLAN标签，就能实现网络的隔离。

```bash
# 设置vnet0的VLAN tag为100
ovs-vsctl set Port vnet0 tag=100
# 设置vnet1和vnet2的VLAN tag为200
ovs-vsctl set Port vnet1 tag=200
ovs-vsctl set Port vnet2 tag=200
```

使用`ovs-vsctl show`命令查看VLAN tag是否配置成功

```bash
90139c71-8d11-49b2-b44c-f34174259dc8
    Bridge br-int
        Port "vnet0"
            tag: 100
            Interface "vnet0"
                type: internal
        Port br-int
            Interface br-int
                type: internal
        Port "vnet2"
            tag: 200
            Interface "vnet2"
                type: internal
        Port "vnet1"
            tag: 200
            Interface "vnet1"
                type: internal
    ovs_version: "2.9.0"
```

测试

测试`ns0`与`ns1`的能否通信 `ip netns exec ns0 ping 10.0.0.2`

```bash
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
^C
--- 10.0.0.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1000ms
```

测试`ns0`与`ns2`的能否通信 `ip netns exec ns0 ping 10.0.0.3`

```bash
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
^C
--- 10.0.0.3 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 999ms
```

测试`ns1`与`ns2`的能否通信 `ip netns exec ns1 ping 10.0.0.3`

```bash
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
64 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=0.930 ms
64 bytes from 10.0.0.3: icmp_seq=2 ttl=64 time=0.057 ms
64 bytes from 10.0.0.3: icmp_seq=3 ttl=64 time=0.056 ms
64 bytes from 10.0.0.3: icmp_seq=4 ttl=64 time=0.057 ms
^C
--- 10.0.0.3 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3000ms
rtt min/avg/max/mdev = 0.056/0.275/0.930/0.378 ms
```

测试`ns2`与`ns1`的能否通信 `ip netns exec ns2 ping 10.0.0.2`

```bash
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.088 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.057 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.050 ms
64 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=0.060 ms
^C
--- 10.0.0.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 2999ms
rtt min/avg/max/mdev = 0.050/0.063/0.088/0.017 ms
```

根据测试结果可以看出，`ns0`是无法访问到`ns1`和`ns2`的，`ns1`和`ns2`可以互相访问。这是因为端口`vnet0`的数据报文发出后被OVS修改了包头，增加了VLAN `100`标签，与`vnet1`、`vnet2`的VLAN `200`标签不匹配，OVS交换机便不再将`vnet0`的数据报文发送给其他两个端口，由此便实现了网络隔离。

清理实验环境

```bash
ovs-vsctl del-br br-int
ip netns del ns0
ip netns del ns1
ip netns del ns2
```


