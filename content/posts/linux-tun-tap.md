---
title: "Linux tun/tap 详解"
categories: [ "Linux网络" ]
tags: [ "linux","network","tun/tap" ]
draft: false
slug: "linux-tun-tap"
date: "2020-11-08 22:11:00"
---

> 在计算机网络中，**tun**与**tap**是操作系统内核中的虚拟网络设备。不同于普通靠硬件网络适配器实现的设备，这些虚拟的网络设备全部用软件实现，并向运行于操作系统上的软件提供与硬件的网络设备完全相同的功能。

### tun/tap是什么？

tun是网络层的虚拟网络设备，可以收发第三层数据报文包，如IP封包，因此常用于一些点对点IP隧道，例如OpenVPN，IPSec等。

tap是链路层的虚拟网络设备，等同于一个以太网设备，它可以收发第二层数据报文包，如以太网数据帧。Tap最常见的用途就是做为虚拟机的网卡，因为它和普通的物理网卡更加相近，也经常用作普通机器的虚拟网卡。

### 如何操作tun/tap？

Linux tun/tap可以通过网络接口和字符设备两种方式进行操作。

当应用程序使用标准网络接口socket API操作tun/tap设备时，和操作一个真实网卡无异。

当应用程序使用字符设备操作tun/tap设备时，字符设备即充当了用户空间和内核空间的桥梁直接读写二层或三层的数据报文。在 Linux 内核 2.6.x 之后的版本中，tun/tap 对应的字符设备文件分别为：

```bash
tun：/dev/net/tun
tap：/dev/tap0
```

当应用程序打开字符设备时，系统会自动创建对应的虚拟设备接口，一般以tunX和tapX方式命名，虚拟设备接口创建成功后，可以为其配置IP、MAC地址、路由等。当一切配置完毕，应用程序通过此字符文件设备写入IP封包或以太网数据帧，tun/tap的驱动程序会将数据报文直接发送到内核空间，内核空间收到数据后再交给系统的网络协议栈进行处理，最后网络协议栈选择合适的物理网卡将其发出，到此发送流程完成。而物理网卡收到数据报文时会交给网络协议栈进行处理，网络协议栈匹配判断之后通过tun/tap的驱动程序将数据报文原封不动的写入到字符设备上，应用程序从字符设备上读取到IP封包或以太网数据帧，最后进行相应的处理，收取流程完成。

> 注意：当应用程序关闭字符设备时，系统也会自动删除对应的虚拟设备接口，并且会删除掉创建的路由等信息。

### tun/tap的区别

tun/tap 虽然工作原理一致，但是工作的层次不一样。

tun是三层网络设备，收发的是IP层数据包，无法处理以太网数据帧，例如OpenVPN的路由模式就是使用了tun网络设备，OpenVPN Server重新规划了一个网段，所有的客户端都会获取到该网段下的一个IP，并且会添加对应的路由规则，而客户端与目标机器产生的数据报文都要经过OpenVPN网关才能转发。

tap是二层网络设备，收发以太网数据帧，拥有MAC层的功能，可以和物理网卡通过网桥相连，组成一个二层网络。例如OpenVPN的桥接模式可以从外部打一条隧道到本地网络。进来的机器就像本地的机器一样参与通讯，丝毫看不出这些机器是在远程。如果你有使用过虚拟机的经验，桥接模式也是一种十分常见的网络方案，虚拟机会分配到和宿主机器同网段的IP，其他同网段的机器也可以通过网络访问到这台虚拟机。

### 使用方式

Linux 提供了一些命令行程序方便我们来创建持久化的tun/tap设备，但是如果没有应用程序打开对应的文件描述符，tun/tap的状态一直会是DOWN，还好的是这并不会影响我们把它当作普通网卡去使用。

使用`ip tuntap help`查看使用帮助

```bash
Usage: ip tuntap { add | del | show | list | lst | help } [ dev PHYS_DEV ]
	[ mode { tun | tap } ] [ user USER ] [ group GROUP ]
	[ one_queue ] [ pi ] [ vnet_hdr ] [ multi_queue ] [ name NAME ]

Where:	USER  := { STRING | NUMBER }
	GROUP := { STRING | NUMBER }
```

### 示例

```bash
# 创建 tap 
ip tuntap add dev tap0 mode tap 
# 创建 tun
ip tuntap add dev tun0 mode tun 

# 删除 tap
ip tuntap del dev tap0 mode tap
# 删除 tun
ip tuntap del dev tun0 mode tun 
```

tun/tap 设备创建成功后可以当作普通的网卡一样使用，因此我们也可以通过`ip link`命令来操作它。

``` shell
# 例如使用ip link命令也可以删除tun/tap设备
ip link del tap0
ip link del tun0
```