---
title: "Linux虚拟化技术KVM"
categories: [ "虚拟化" ]
tags: [ "linux","kvm" ]
draft: false
slug: "linux-kvm"
date: "2021-05-29T17:07:00+08:00"
---

在Windows平台上我们习惯于使用VmWare或者virtual box来实现虚拟化，虽然它们拥有Linux版本，但大多数企业都选择了使用KVM来做Linux平台的虚拟化，因此学习掌握KVM是一项必不可少的技能。

### 安装KVM

以centos为例，下面是安装KVM虚拟化的命令。

```
yum install -y qemu-kvm libvirt virt-install bridge-utils
```

**这么多软件都是什么作用？**

| 软件         | 作用                                                         |
| ------------ | ------------------------------------------------------------ |
| qemu-kvm     | 整合了QEMU 和 KVM 的一个软件。                               |
| libvirt      | 封装了QEMU的接口，可以更加方便的操作虚拟机，并且提供了很多种编程语言的SDK。 |
| virt-install | 用来创建虚拟机的命令行工具。                                 |
| bridge-utils | Linux网桥，用来配置虚拟机的桥接网络。                        |

**kvm、qemu、qemu-kvm和libvirt到底有什么关系？**

KVM（Kernel Virtual Machine）是Linux的一个内核驱动模块，它需要CPU的支持，采用硬件辅助虚拟化技术Intel-VT、AMD-V；内存相关如Intel的EPT和AMD的RVI技术，使得它能够让Linux主机成为一个Hypervisor（虚拟机监控器）。

QEMU是一个纯软件实现的虚拟机，它可以模拟CPU、内存、磁盘等其他硬件，让虚拟机认为自己底层就是硬件，其实这些都是QEMU模拟的，虚拟机的所有操作都要经过QEMU转译一层，也就导致了QEMU本身的性能较差。

qemu-kvm是QEMU整合了KVM，把CPU虚拟化和内存虚拟化交给了KVM来做，自己来模拟IO设备，例如网卡和磁盘。这一套组合拳打下来，性能损失大大降低，相较于直接使用硬件，带来的损耗大概在1%-2%之间。

libvirt是目前使用最为广泛的对KVM虚拟机进行管理的工具和API。Libvirtd是一个daemon进程，可以被本地的virsh调用，也可以被远程的virsh调用，Libvirtd调用qemu-kvm操作虚拟机。

**启动libvirt**

```bash
systemctl start libvirtd
systemctl enable libvirtd
```

如果你不想使用命令行工具来管理虚拟机，可以安装 virt-manager 。

```bash
yum install -y virt-manager
```

在支持x11转发的ssh客户端（例如：[MobaXterm](https://mobaxterm.mobatek.net/)）上可以直接输入 virt-manager 来启动。

### 虚拟网络类型

和vmware类型，kvm也支持多种类型的网络，主要分为三种。

1. **NAT模式** 虚拟机需要把流量发送到宿主机，宿主机器转换网络信息后再发出，外部机器无法感知到虚拟机的存在。此种方式宿主机器相当于一个路由器，因此宿主机上会有一个和虚拟机同网段的IP，并且虚拟机的网关地址是宿主机的这个IP。

2. **主机模式** 虚拟机只能互相访问，不能访问宿主机。此种方式与NAT模式类似，但它没有与虚拟机同网段的IP，因此虚拟机也不能借助于宿主机来访问外部网络。

3. **桥接模式** 虚拟机和宿主机都关联在一个网桥上，因此虚拟机可以与宿主机在同一个网段，并且外部机器可以直接访问到虚拟机，虚拟机也可以借助网桥来访问外部网络。

   > 还有一种模式在openstack等云平台上使用较为广泛，网桥上绑定的物理网卡没有IP，对应交换机配置端口为trunk模式，虚拟机 端口连接到网桥上，并配置端口不同的VLAN tag以达到隔离和互联的目的。

NAT模式和主机模式都无需单独配置，接下来我们看下如何配置桥接网络。

### 配置桥接网络

物理网卡绑定到网桥上之后就会导致网络断开，因此我们需要把原IP配置到网桥上。

```
# 进入网卡配置文件夹
cd /etc/sysconfig/network-scripts/
# 拷贝原网卡配置文件作为桥接网卡
cp ifcfg-enp134s0f0 ifcfg-br0
```

修改 `ifcfg-br0` 中的 `TYPE=Ethernet`  为 ` TYPE=Bridge`，最终效果如下：

```bash
DEVICE=br0
ONBOOT=yes
BOOTPROTO=none
TYPE=Bridge
IPADDR=172.16.0.52
PREFIX=16
GATEWAY=172.16.0.1
DNS1=114.114.114.114
```

修改`ifcfg-enp134s0f0` 文件删除其中的 `IPADDR=`  `NETMASK=`  `GATEWAY=` 行，并在最后添加上`BRIDGE=br0`，最终效果如下：

```bash
DEVICE="enp134s0f0"
ONBOOT=yes
BOOTPROTO=none
TYPE=Ethernet
BRIDGE=br0
```

最后重启网络。

```bash
systemctl restart network
```

查看网络信息也可以看到IP配置到了网桥上，网桥上又关联了物理网卡。

```bash
[root@localhost network-scripts]# ip a
8: enp134s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master br0 state UP group default qlen 1000
    link/ether 74:a4:b5:01:04:22 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::76a4:b5ff:fe01:422/64 scope link 
       valid_lft forever preferred_lft forever
17: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 74:a4:b5:01:04:22 brd ff:ff:ff:ff:ff:ff
    inet 172.16.0.52/16 brd 172.16.255.255 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::76a4:b5ff:fe01:422/64 scope link 
       valid_lft forever preferred_lft forever
[root@localhost network-scripts]# brctl show
bridge name     bridge id               STP enabled     interfaces
br0             8000.74a4b5010422       no              enp134s0f0
```

### 创建虚拟机

创建虚拟机有三种选择

1. **virt-install** 使用命令创建虚拟机较为方便快捷，但对新手不友好，很多参数不知道如何设置。
2. **Libvirt** 使用libvirt提供的api接口进行创建，此种方式开发虚拟化平台会用到。
3. **virt-manager** 使用图形化界面创建虚拟机，此种方式较为简单，只要会用vmware，就会用virt-manager。

因此我比较推荐使用**virt-manager**来创建虚拟机，过程很简单，我就不发截图了。

### 管理虚拟机

操作虚拟机的一些常用命令

```bash
# 列出正在运行的虚拟机
virsh list
# 列出全部的虚拟机
virsh list --all
# 启动虚拟机
virsh start test  
# 关闭虚拟机
virsh shutdown test  
# 强制停止虚拟机
virsh destroy test  
# 彻底销毁虚拟机，会删除虚拟机配置文件，但不会删除虚拟磁盘
virsh undefine test  
# 设置宿主机开机时该虚拟机也开机
virsh autostart test  
# 解除开机启动
virsh autostart --disable test 
# 挂起虚拟机
virsh suspend test 
# 恢复挂起的虚拟机
virsh resume test 
# 查看虚拟机的VNC端口，得到vnc端口之后可以使用vncviewer等工具访问虚拟机
virsh vncdisplay test
# 导出虚拟机xml配置文件
virsh dumpxml test >/root/test.xml
# 修改虚拟机配置文件
virsh edit test
```

### 参考

https://blog.51cto.com/changfei/1672147

