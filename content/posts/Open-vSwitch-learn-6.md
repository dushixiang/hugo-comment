---
title: "Open vSwitch 入门实践（6）VXLAN实验"
categories: [ "Open vSwitch" ]
tags: [ "sdn","ovs" ]
draft: false
slug: "ovs-learn-6"
date: "2020-12-30 19:14:00"
---

# 什么是VXLAN？

VXLAN是一种隧道封装协议，在三层网络上封装二层网络数据报文。简单来说就是可以在已经规划好网络拓扑的设备上封装出一个新的二层网络，因此VXLAN这类网络又被称之为overylay网络，底下承载VXLAN网络的就被称之为underlay网络。

# VXLAN解决了什么问题？

最近几年，阿里云，腾讯云，京东云，华为云等等厂商每到节日都会打折出售大量云服务器，1核1G内存50G磁盘的服务器几十块就能买到一年的使用权，作为一个专业的羊毛党，哪个手里没有几台小破水管机器？但是这么多的云服务器是厂商如何做隔离的呢？了解过网络的同学或许会说VLAN。但是VLAN这种只能隔离4094个虚拟网络的技术别说满足不了羊毛党了，就连正常的用户估计都撑不住。那不隔离能行吗，厂商规划一个特别大的网段，让大家都在这里面耍，正常用户还好，万一这个时候进来一个大黑客，估计就会全部GG。

因此，隔离是必不可少的，其中关键的技术就是overlay网络。

那VXLAN具体解决了哪些问题呢？

- 突破了VLAN技术4094个隔离网络的限制，在一个管理域中创建多达1600万个VXLAN网络。
- VXLAN提供了云服务厂商所需的规模的网络分段，以支持大量租户。
- 突破了物理网络边界的限制，传统虚拟二层网络（VLAN）是需要和物理网络做大量适配工作才能保证环境的迁移不会导致虚拟网络异常，overlay网络则不必关心底层物理网络是如何搭建的，只要能保证VXLAN端点相互之间可以联通即可。

# VXLAN网络如何工作？

VXLAN隧道协议将二层以太网帧封装在三层UDP数据包中，使用户能够创建跨物理三层网络的虚拟化二层子网或网段。每个二层子网使用VXLAN网络标识符（VNI）作为唯一标识。报文格式如下图：

![VXLAN报文格式](https://oss.typesafe.cn/vxlan_packet_header.png)

执行数据包封装和解封装的实体称为VXLAN隧道终结点（VTEP）。VTEP主要分为两类：硬件VTEP和软件VTEP。硬件VTEP我接触较少，这里就不再介绍了。

软件VTEP如下图所示：VTEP在数据包到达虚拟机之前进行了封装和解封装，使得虚拟机完全不需要知道VXLAN隧道以及它们之间的三层网络。


![vxlan网络](https://oss.typesafe.cn/vxlan01.png)

# 简单VXLAN实验

我们参照下图完成实验。

![VXLAN实验](https://oss.typesafe.cn/vxlan_topo.png)

### 主机A
```
# 创建隧道网桥
ovs-vsctl add-br br-tun
# 创建隧道端口并指定远端IP和VXLAN ID
ovs-vsctl add-port br-tun vx01 -- set Interface vx01 type=vxlan options:remote_ip=192.168.123.232 options:key=1111
# 创建内部端口
ovs-vsctl add-port br-tun vnet0 -- set Interface vnet0 type=internal
# 创建netns用于模拟虚拟网络设备
ip netns add ns0
# 将内部端口移动到netns中
ip link set vnet0 netns ns0
# 启动网卡
ip netns exec ns0 ip link set lo up
ip netns exec ns0 ip link set vnet0 up
# 配置IP
ip netns exec ns0 ip addr add 192.168.0.1/24 dev vnet0
```

### 主机B
```
# 创建隧道网桥
ovs-vsctl add-br br-tun
# 创建隧道端口并指定远端IP和VXLAN ID
ovs-vsctl add-port br-tun vx01 -- set Interface vx01 type=vxlan options:remote_ip=192.168.123.231 options:key=1111
# 创建内部端口
ovs-vsctl add-port br-tun vnet0 -- set Interface vnet0 type=internal
# 创建netns用于模拟虚拟网络设备
ip netns add ns0
# 将内部端口移动到netns中
ip link set vnet0 netns ns0
# 启动网卡
ip netns exec ns0 ip link set lo up
ip netns exec ns0 ip link set vnet0 up
# 配置IP
ip netns exec ns0 ip addr add 192.168.0.2/24 dev vnet0
```

## 测试

在`主机A`上测试网络连通性 `ip netns exec ns0 ping 192.168.0.2`

```
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=0.715 ms
64 bytes from 192.168.0.2: icmp_seq=2 ttl=64 time=0.372 ms
64 bytes from 192.168.0.2: icmp_seq=3 ttl=64 time=0.205 ms
64 bytes from 192.168.0.2: icmp_seq=4 ttl=64 time=0.230 ms
^C
--- 192.168.0.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 111ms
rtt min/avg/max/mdev = 0.205/0.380/0.715/0.204 ms
```

可以看到我们只使用一张网卡就成功的在已有的物理网络上建立了一个新的分布式二层网络。

> 在这个实验中我使用了固定的VXLAN ID 1111，只是为了简化实验，无其他含义。

当我们此时在主机B上对物理网卡`eth0`使用`tcpdump`抓包时。

```shell
tcpdump -l -n -vv -i eth0 'port 4789 and udp[8:2] = 0x0800 & 0x0800 and udp[11:4] = 1111 & 0x00FFFFFF'
```

可以看到类似我下面截取的数据包。

```shell
09:09:59.080604 IP (tos 0x0, ttl 64, id 12981, offset 0, flags [DF], proto UDP (17), length 134)
    192.168.123.231.59648 > 192.168.123.232.vxlan: [no cksum] VXLAN, flags [I] (0x08), vni 1111
IP (tos 0x0, ttl 64, id 29815, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.0.1 > 192.168.0.2: ICMP echo request, id 1928, seq 1, length 64
```

这是一条完整的数据报文，只是格式化成了4行。

前两行是`underlay`网络的报文，也就是底层承载网络。分别是：
- 底层协议 `proto UDP (17)`
- 包长度 `length 134`
- 源和目的地址 `192.168.123.231.59648 > 192.168.123.232.vxlan`
- VXLAN ID：`vni 1111`

后两行是`overlay`网络的报文，也就是我们的虚拟网络。分别是：
- 承载的协议是 `proto ICMP (1)`
- 包长度 `length 84`
- 源和目的地址 `192.168.0.1 > 192.168.0.2`

可以看出这些信息和我们的实验环境非常匹配，只有包大小不太一致，这是因为`underlay`网络的基础信息刚好占有了50个字节，不信你可以加一下上面的报文格式中的数字。

最后也不要忘记清理实验环境。


```bash
# 在主机A和B上都需要执行
ovs-vsctl del-br br-tun
ip netns del ns0
```