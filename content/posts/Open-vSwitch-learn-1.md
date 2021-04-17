---
title: "Open vSwitch 入门实践（1）Open vSwitch 是什么"
categories: [ "Open vSwitch" ]
tags: [ "sdn","ovs" ]
draft: false
slug: "ovs-learn-1"
date: "2020-11-25 17:49:00"
---

# OVS简介

### Open vSwitch 是什么？

Open vSwitch(以下简称OVS)是一个用C语言开发的多层虚拟交换机，使用Apcahe 2开源许可证，现如今基本上已经成为了开源SDN（软件定义网络）基础设施层的事实标准。

### OVS支持哪些[功能](http://www.openvswitch.org//features/)？

- 支持NetFlow、sFlow(R)、IPFIX、SPAN、RSPAN和GRE隧道镜像等多种流量监控协议
- 支持LACP (IEEE 802.1AX-2008)
- 支持标准802.1Q VLAN协议，允许端口配置trunk模式
- 支持组播
- 支持BFD和802.1ag链路监控
- 支持STP（IEEE 802.1D-1998）和RSTP（IEEE 802.1D-2004）
- 支持细粒度的QoS（服务质量）配置
- 支持HFSC qdisc
- 支持接管每一个虚拟机的流量
- 支持基于源MAC的负载均衡、主备模式和L4哈希的端口绑带
- 支持OpenFlow协议（包含了很多对虚拟化的扩展）
- 支持IPv6
- 支持多种隧道协议（GRE、VXLAN、STT、Geneve和IPsec）
- 支持C和Python的远程配置协议
- 支持内核和用户空间的转发引擎选项
- 具有流缓存引擎的多表转发管道
- 转发层抽象以简化向新软件和硬件平台的移植

# OVS的术语解释

### Bridge

中文名称**网桥**，一个Bridge代表一个以太网交换机（Switch），一台主机中可以创建一个或多个Bridge，Bridge可以根据一定的规则，把某一个端口接收到的数据报文转发到另一个或多个端口上，也可以修改或者丢弃数据报文。

### Port

中文名称**端口**，需要注意的是它和TCP里面的端口不是同样的概念，它更像是物理交换机上面的插口，可以接水晶头的那种。Port隶属于Bridge，必须先添加了Bridge才能在Bridge上添加Port。Port有以下几种类型：

- **Normal**

  用户可以把操作系统中已有的网卡添加到Open vSwicth上，Open vSwitct会自动生成一个同名的Port开处理这张网卡进和出的数据报文。

  >  不过需要注意的是这种方式添加的Port不支持分配IP地址，如果之前网卡上配置的有IP，挂载到OVS上面之后将不可访问。此类型的Port常用于VLAN模式的多台物理主机相连的那个口，交换机一端属于Trunk模式。

- **Internal**

  当Port的类型是Internal时，OVS会自动创建一个虚拟网卡（Interface），此端口收到的数据报文都会转发给这块网卡，从这块网卡发出的数据报文也会通过Port交给OVS处理。当OVS创建一个新的网桥时，会自动创建一个与网桥同名的Internal Port，同时也会创建一个与网桥同名的Interface，因此可以通过ip命令在操作系统中查看到这张虚拟网卡，但是状态是down的。

- **Patch**

  Patch Port和veth pair功能相同，总是成双成对的出现，在其中一端收到的数据报文会被转发到另一个Patch Port上，就像是一根网线一样。Patch Port常用于连接两个Bridge，这样两个网桥就和一个网桥一样了。


- **Tunnel**

  OVS 支持 GRE、VXLAN、STT、Geneve和IPsec隧道协议，这些隧道协议就是overlay网络的基础协议，通过对物理网络做的一层封装和扩展，解决了二层网络数量不足的问题，最大限度的减少对底层物理网络拓扑的依赖性，同时也最大限度的增加了对网络的控制。

### Interface

（iface/接口）接口是OVS与操作系统交换数据报文的组件，一个接口即是操作系统上的一块网卡，这个网卡可能是OVS生成的虚拟网卡，也有可能是挂载在OVS上的物理网卡，操作系统上的虚拟网卡（TUN/TAP）也可以被挂载在OVS上。

### Controller

OpenFlow控制器，OVS可以接收一个或者多个OpenFlow控制器的管理，功能主要是下发流表，控制转发规则。

### Flow

流表是OVS进行数据转发的核心功能，定义了端口之间转发数据报文的规则，一条流表规则主要分为匹配和动作两部分，匹配部分决定哪些数据报文需要被处理，动作决定了匹配到的数据报文该如何处理。

# OVS常用操作

### 安装

```bash
yum install openvswitch
systemctl enable openvswitch
systemctl start openvswitch
```

> 如果当前软件源中没有openvswitch，可以通过[阿里云官方镜像站](https://developer.aliyun.com/packageSearch?word=openvswitch)下载和操作系统版本对应的rpm包到本地再安装。 示例命令： `yum localinstall openvswitch-2.9.0-3.el7.x86_64.rpm` 

### Bridge 操作

添加网桥

```bash
ovs-vsctl add-br br-int
```

查询网桥列表

```bash
ovs-vsctl list-br
```

删除网桥

```bash
ovs-vsctl del-br br-int
```

### Port 操作

- **Normal Port**

```bash
# 将物理网卡eth0添加到网桥br-int上
ovs-vsctl add-port br-int eth0
# 移除网桥br-int上的Port
ovs-vsctl del-port br-int eth0
```

- **Internal Port**

```bash
# 添加Internal Port 
ovs-vsctl add-port br-int vnet0 -- set Interface vnet0 type=internal
# 把网卡vnet0启动并配置IP
ip link set vnet0 up
ip addr add 192.168.0.1/24 dev vnet0
# 设置VLAN tag
ovs-vsctl set Port vnet0 tag=100
# 移除vnet0上面的VLAN tag配置
ovs-vsctl remove Port vnet0 tag 100
# 设置vnet0允许通过的VLAN tag
ovs-vsctl set Port vnet0 trunks=100,200
# 移除vnet0允许通过的的VLAN tag配置
ovs-vsctl remove Port vnet0 trunks 100,200
```

- **Patch Port**

```bash
ovs-vsctl add-br br0
ovs-vsctl add-br br1
ovs-vsctl \
-- add-port br0 patch0 -- set interface patch0 type=patch options:peer=patch1 \
-- add-port br1 patch1 -- set interface patch1 type=patch options:peer=patch0
```

- **Tunnel Port**

```bash
#主机10.1.7.21上
ovs-vsctl add-br br-tun
ovs-vsctl add-port br-tun vxlan-vx01 -- set Interface vxlan-vx01 type=vxlan options:remote_ip=10.1.7.22 options:key=flow
ovs-vsctl add-port br-tun vxlan-vx02 -- set Interface vxlan-vx02 type=vxlan options:remote_ip=10.1.7.23 options:key=flow

#主机10.1.7.22上
ovs-vsctl add-br br-tun
ovs-vsctl add-port br-tun vxlan-vx01 -- set Interface vxlan-vx01 type=vxlan options:remote_ip=10.1.7.21 options:key=flow
ovs-vsctl add-port br-tun vxlan-vx02 -- set Interface vxlan-vx02 type=vxlan options:remote_ip=10.1.7.23 options:key=flow

#主机10.1.7.23上
ovs-vsctl add-br br-tun
ovs-vsctl add-port br-tun vxlan-vx01 -- set Interface vxlan-vx01 type=vxlan options:remote_ip=10.1.7.21 options:key=flow
ovs-vsctl add-port br-tun vxlan-vx02 -- set Interface vxlan-vx02 type=vxlan options:remote_ip=10.1.7.22 options:key=flow
```

- 其他基本操作

```bash
# 设置VLAN mode
ovs-vsctl set port <port name> VLAN_mode=trunk|access|native-tagged|native-untagged
# 设置VLAN tag
ovs-vsctl set port <port name> tag=<1-4095>
# 设置VLAN trunk
ovs-vsctl set port <port name> trunk=100,200
# 移除Port的属性
ovs-vsctl remove port <port name> <property name> <property value>
# 查看Port的属性
ovs-vsctl list interface <port name>
```

接下来我们将使用OVS来实现单机和多台物理服务器下的虚拟VLAN网络。