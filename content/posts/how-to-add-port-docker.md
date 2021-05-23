---
title: "容器网络——如何为docker添加网卡？"
categories: [ "docker" ]
tags: [ "docker","network" ]
draft: false
slug: "how-to-add-port-for-docker"
date: "2021-05-23T13:37:00+08:00"
---

之前我们介绍`Network Namespace`（以下简称`netns`）和`veth pair`时说过`docker`是使用这些技术来实现的网络隔离，今天我们就来一探究竟，看下`docker`到底是如何做到的。

### 启动一个无网络的容器

首先我们使用 `--net=none` 参数启动一个无网络的容器，为了方便调试，这里我们使用了`centos`镜像。

``` bash
docker run -itd --name centos-test --net=none centos
```

启动成功之后我们进入容器内部确认一下是否无网卡。

```bash
[root@localhost ~]# docker ps
CONTAINER ID   IMAGE          COMMAND       CREATED          STATUS          PORTS     NAMES
28dc2e8853df   centos         "/bin/bash"   24 seconds ago   Up 23 seconds             centos-test
[root@localhost ~]# docker exec -it 28dc2e8853df bash
[root@28dc2e8853df /]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
```

可以看到确实只有一个本地环回网卡。

### 如何查看docker对应的netns？

当容器启动时，`docker`内部会自动为这个容器创建一个`netns`用于网络隔离，但当我们使用 `ip netns list` 查看时却看不到任何数据，这是因为`docker`把 `netns` 创建在了其他地方，而`ip netns list`命令只能读目录`/var/run/netns`下面的数据。

我们可以通过以下命令来解决这个问题，方便我们学习`docker`网络。

``` bash 
# 得到容器对应的进程
pid=docker inspect -f '{{.State.Pid}}' "$container_id"
# 手动创建防止文件夹不存在
mkdir -p /var/run/netns/
# 建立软连接
ln -s /proc/$pid/ns/net /var/run/netns/$container_id
```

将上面的命令修改为和当前环境一致并验证netns中的网卡。

```bash
[root@localhost ~]# docker inspect -f '{{.State.Pid}}' "28dc2e8853df"
123624
[root@localhost ~]# mkdir -p /var/run/netns/
[root@localhost ~]# ln -s /proc/123624/ns/net /var/run/netns/28dc2e8853df
[root@localhost ~]# ip netns list
28dc2e8853df
[root@localhost ~]# ip netns exec 28dc2e8853df ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
```

可以看到我们成功输出了`netns`中的网卡信息，并且此信息和从`docker`容器中看到的一致。

### 给docker添加网卡

参考前面的`Linux Bridge`章节，我们首先创建一个网桥，然后创建一对`veth pair`，一端连接到网桥，一端移动到`docker`对应的`netns`中。

```bash
# 添加网桥
brctl addbr br0
# 启动网桥
ip link set br0 up
# 新增一对veth
ip link add veth0-ns type veth peer name veth0-br
# 将veth的一端移动到docker对应的netns中
ip link set veth0-ns netns 28dc2e8853df
# 将netns中的本地环回和veth启动并配置IP
ip netns exec 28dc2e8853df ip link set lo up
ip netns exec 28dc2e8853df ip link set veth0-ns up
ip netns exec 28dc2e8853df ip addr add 10.0.0.1/24 dev veth0-ns
# 将veth的另一端启动并挂载到网桥上
ip link set veth0-br up
brctl addif br0 veth0-br
```

最后验证`netns`和`docker`容器中的网卡信息。

```bash
[root@localhost ~]# ip netns exec 28dc2e8853df ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
91: veth0-ns@if90: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 86:29:e6:0a:2a:cb brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.1/24 scope global veth0-ns
       valid_lft forever preferred_lft forever
[root@localhost ~]# docker exec -it 28dc2e8853df bash
[root@28dc2e8853df /]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
91: veth0-ns@if90: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 86:29:e6:0a:2a:cb brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.1/24 scope global veth0-ns
       valid_lft forever preferred_lft forever
```

可以看出我们在`netns`中添加的网卡在`docker`容器中也可以正确的显示出来，由此证明了此`netns`和`docker`容器的对应关系。当我们使用`docker`命令来创建容器时，`docker`为我们隐藏了大量细节，轻松使用几条命令便创建好了容器，但这种只知其然不知其所以然的方式对于我们掌握`docker`并不够，只有了解了底层原理之后才能对其功能掌握的更加深入。

### 理解docker的几种网络模式

了解了`docker`添加网卡的原理后再来理解`docker`的几种网络模式就十分简单明了了，主要区别就在于是否具有独立的`netns`。

| 网络模式  | 简介                                                         |
| --------- | ------------------------------------------------------------ |
| bridge    | 容器具有独立的`netns`，会将容器连接到 `docker0` 虚拟网桥，并配置IP地址，默认为该模式。 |
| host      | 容器没有独立的`netns`，和宿主机共用网络。                    |
| none      | 容器具有独立的`netns`，但并没有对其进行任何网络设置。        |
| container | 容器和某一个已存在的容器共享`netns`。                        |

> 新版docker新增了ipvlan、macvlan和overlay类型的网络，主要是为了多台宿主机器上面的docker容器隔离与通信，底层网络复杂了很多，之后我们会单独对其进行介绍。

最后也不要忘记清理实验环境哦。

```bash
# 删除网桥
ip link del br0
# 删除veth pair
ip link del veth0-br
# 删除软连接
rm -rf /var/run/netns/28dc2e8853df
# 删除容器
docker rm 28dc2e8853df -f
```

