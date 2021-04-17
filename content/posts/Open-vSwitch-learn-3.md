---
title: "Open vSwitch 入门实践（3）使用OVS构建分布式隔离网络"
categories: [ "Open vSwitch" ]
tags: [ "sdn","ovs" ]
draft: false
slug: "ovs-learn-3"
date: "2020-12-27 11:38:00"
---

# 使用OVS构建分布式隔离网络

## 前言

上一节我们使用OVS构建了单机隔离网络，但是随着网络规模的扩张，单节点已经不再能满足业务的需要，分布式网络成了必不可少的环节。分布式网络与单节点网络在细节实现上基本一致，只有物理环境网络连线上的一点区别。

## 实验1：分布式无隔离网络

网络拓扑如下图所示，我们每一台节点都有两张网卡，一张用于管理，一张用于业务。之所以使用两张网卡有两个原因：
1. 管理网卡用于日常的维护登录，业务网卡用于传输虚拟节点的数据报文，避免相互之间影响。
2. 我们要将业务网卡绑定到OVS网桥上，也就是`Normal`类型的`Port`。这种方式添加的`Port`不支持分配IP地址，如果之前网卡上配置的有IP，挂载到OVS上面之后将不可访问。

> 需要注意的是，如果是使用物理环境搭建网络拓扑，需要把业务网卡对应的交换机端口配置为`trunk`模式。如果是使用VmWare搭建网络拓扑，业务网卡需要配置网络类型为`仅主机模式`。

![分布式无隔离网络](https://oss.typesafe.cn/ovs-di-network0.png?t=2)

### 配置
- 配置环境 `主机A`

``` bash
ovs-vsctl add-br br-int
# 请修改eth1为当前实验环境的业务网卡名称
ovs-vsctl add-port br-int eth1

# 添加两个内部端口
ovs-vsctl add-port br-int vnet0 -- set Interface vnet0 type=internal
ovs-vsctl add-port br-int vnet1 -- set Interface vnet1 type=internal
# 添加两个netns
ip netns add ns0
ip netns add ns1
# 将内部端口分别移动到netns中
ip link set vnet0 netns ns0
ip link set vnet1 netns ns1

# 启动端口并配置IP
ip netns exec ns0 ip link set lo up
ip netns exec ns0 ip link set vnet0 up
ip netns exec ns0 ip addr add 10.0.0.1/24 dev vnet0

ip netns exec ns1 ip link set lo up
ip netns exec ns1 ip link set vnet1 up
ip netns exec ns1 ip addr add 10.0.0.2/24 dev vnet1
```

- 配置环境 `主机B`

``` bash
ovs-vsctl add-br br-int
# 请修改eth1为当前实验环境的业务网卡名称
ovs-vsctl add-port br-int eth1

# 添加两个内部端口
ovs-vsctl add-port br-int vnet0 -- set Interface vnet0 type=internal
ovs-vsctl add-port br-int vnet1 -- set Interface vnet1 type=internal
# 添加两个netns
ip netns add ns0
ip netns add ns1
# 将内部端口分别移动到netns中
ip link set vnet0 netns ns0
ip link set vnet1 netns ns1

# 启动端口并配置IP
ip netns exec ns0 ip link set lo up
ip netns exec ns0 ip link set vnet0 up
ip netns exec ns0 ip addr add 10.0.0.3/24 dev vnet0

ip netns exec ns1 ip link set lo up
ip netns exec ns1 ip link set vnet1 up
ip netns exec ns1 ip addr add 10.0.0.4/24 dev vnet1
```

### 测试

- 测试 `主机A`

``` bash
ip netns exec ns0 ping 10.0.0.3
ip netns exec ns0 ping 10.0.0.4
ip netns exec ns1 ping 10.0.0.3
ip netns exec ns1 ping 10.0.0.4
```


- 测试 `主机B`

``` bash
ip netns exec ns0 ping 10.0.0.1
ip netns exec ns0 ping 10.0.0.2
ip netns exec ns1 ping 10.0.0.1
ip netns exec ns1 ping 10.0.0.2
```

- 测试结果

| 主机A | 主机B | ping 结果 |
| --- | --- | --- |
| ns0 | ns0 | 可通信 ✅ |
| ns0 | ns1 | 可通信 ✅ |
| ns1 | ns0 | 可通信 ✅ |
| ns1 | ns1 | 可通信 ✅ |

根据测试结果可以看到我们使用OVS成功的联通了分布在不同主机上的虚拟网络设备。

## 实验2：分布式隔离网络

构建分布式隔离网络和单节点的操作方法一致，即给对应的端口配置VLAN tag。如下图所示，我们分别给主机A、B上的端口配置VLAN tag为100和200。

![分布式无隔离网络](https://oss.typesafe.cn/ovs-di-network1.png?t=2)

### 配置

- 配置环境 `主机A`

``` bash
ovs-vsctl set Port vnet0 tag=100
ovs-vsctl set Port vnet1 tag=200
```

- 配置环境 `主机B`

``` bash
ovs-vsctl set Port vnet0 tag=100
ovs-vsctl set Port vnet1 tag=200
```

### 测试

- 测试 `主机A`

``` bash
ip netns exec ns0 ping 10.0.0.3
ip netns exec ns0 ping 10.0.0.4
ip netns exec ns1 ping 10.0.0.3
ip netns exec ns1 ping 10.0.0.4
```


- 测试 `主机B`

``` bash
ip netns exec ns0 ping 10.0.0.1
ip netns exec ns0 ping 10.0.0.2
ip netns exec ns1 ping 10.0.0.1
ip netns exec ns1 ping 10.0.0.2
```

- 测试结果

| 主机A | 主机B | ping 结果 |
| --- | --- | --- |
| ns0 | ns0 | 可通信 ✅ |
| ns0 | ns1 | 不通信 ❌ |
| ns1 | ns0 | 不通信 ❌|
| ns1 | ns1 | 可通信 ✅ |

根据测试结果可以看到我们使用OVS成功的隔离了分布在不同主机上的虚拟网络设备。