---
title: "Open vSwitch 入门实践（4）使用OVS配置端口镜像"
categories: [ "Open vSwitch" ]
tags: [ "sdn","ovs" ]
draft: false
slug: "ovs-learn-4"
date: "2020-12-28 19:23:00"
---

# 前言

当我们想要在不影响虚拟网络设备数据报文收发的情况下获取对应虚拟网络设备的流量时，端口镜像是一个很好的选择。端口镜像是指将经过指定端口（镜像端口）的报文复制一份到另一个指定端口（观察端口），通过观察端口接收到的数据报文，就可以有效识别虚拟网络的运行情况。

OVS提供了相关命令来配置或删除端口镜像，下面我们来实验一下。

# 如何使用

### 端口镜像类型

端口镜像分为镜像源和镜像目的两部分。

#### 镜像源

- select_all：布尔类型（true，false）。设置为 true 时，表示此网桥上的所有流量。
- select_dst_port：字符串（端口名称）。表示此端口接收的所有流量。
- select_src_port：字符串（端口名称）。表示此端口发送的所有流量。
- select_vlan：整型（0-4095）。表示携带此VLAN标签的流量。

#### 镜像目的

- output_port：字符串（端口名称）。接收流量报文的观察端口。
- output_vlan：整型（0-4095）。表示只修改VLAN标签，原VLAN标签会被剥离。

### 基础操作命令

新增端口镜像

```bash
ovs-vsctl -- set Bridge <bridge_name> mirrors=@m \
 -- --id=@<port0> get Port <port0> \
 -- --id=@<port1> get Port <port1> \
 -- --id=@m create Mirror name=<mirror_name> select-dst-port=@<port0> select-src-port=@<port0> output-port=@<port1>
```

> 这行命令会输出一个镜像ID

删除端口镜像

```
ovs-vsctl remove Bridge <bridge-name> mirrors <mirror-id>
```

在原端口镜像的基础上增加一个镜像源

```bash
# 获取端口的ID
ovs-vsctl get port <port_name> _uuid

# 在原端口镜像的基础上增加镜像源
ovs-vsctl add Mirror <mirror-name> select_src_port <port-id>
ovs-vsctl add Mirror <mirror-name> select_dst_port <port-id>
```

在原端口镜像的基础上删除一个镜像源

```bash
# 获取端口的ID
ovs-vsctl get port <port_name> _uuid

ovs-vsctl remove Mirror <mirror-name> select_src_port <port-id>
ovs-vsctl remove Mirror <mirror-name> select_dst_port <port-id>
```

清空端口镜像

```bash
ovs-vsctl clear Mirror 
```

查看端口镜像

```bash
ovs-vsctl list Mirror 
```

关闭端口的MAC地址学习

```bash
ovs-ofctl mod-port <bridge-name> <port-name> NO-FLOOD
```

# 实验

### 实验拓扑

实验拓扑分为一个网桥，三个虚拟网络设备，

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
# 注意这里只启动了网卡，但没有配置IP
ip netns exec ns2 ip link set lo up
ip netns exec ns2 ip link set vnet2 up

ovs-vsctl -- set Bridge br-int mirrors=@m \
 -- --id=@vnet1 get Port vnet1 \
 -- --id=@vnet2 get Port vnet2 \
 -- --id=@m create Mirror name=mirror_test select-dst-port=@vnet1 select-src-port=@vnet1 output-port=@vnet2
```

### 测试

执行以下命令产生流量

```bash
ip netns exec ns0 ping 10.0.0.2
```

重新打开一个终端执行以下命令抓包

```bash
ip netns exec ns2 tcpdump -i vnet2
```

> 需要安装tcpdump才能使用

输出

```bash
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on vnet2, link-type EN10MB (Ethernet), capture size 262144 bytes
22:26:31.140974 IP 10.0.0.1 > 10.0.0.2: ICMP echo request, id 4599, seq 23, length 64
22:26:31.140996 IP 10.0.0.2 > 10.0.0.1: ICMP echo reply, id 4599, seq 23, length 64
22:26:32.141066 IP 10.0.0.1 > 10.0.0.2: ICMP echo request, id 4599, seq 24, length 64
22:26:32.141085 IP 10.0.0.2 > 10.0.0.1: ICMP echo reply, id 4599, seq 24, length 64
22:26:33.141066 IP 10.0.0.1 > 10.0.0.2: ICMP echo request, id 4599, seq 25, length 64
22:26:33.141108 IP 10.0.0.2 > 10.0.0.1: ICMP echo reply, id 4599, seq 25, length 64
22:26:34.141044 IP 10.0.0.1 > 10.0.0.2: ICMP echo request, id 4599, seq 26, length 64
22:26:34.141062 IP 10.0.0.2 > 10.0.0.1: ICMP echo reply, id 4599, seq 26, length 64
^C
8 packets captured
8 packets received by filter
0 packets dropped by kernel
```



### 清理实验环境

``` bash
ip netns del ns0
ip netns del ns1
ip netns del ns2

ovs-vsctl del-br br-int
```

