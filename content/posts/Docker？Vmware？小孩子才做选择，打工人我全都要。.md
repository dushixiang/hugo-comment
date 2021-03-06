---
title: "Docker？Vmware？小孩子才做选择，打工人我全都要。"
categories: [ "tool" ]
tags: [ "docker","vmware" ]
draft: false
slug: "vctl"
date: "2020-11-17 15:39:00"
---

# 前言
作为一个称职的打工人，电脑上常备一个Vmware不是什么新鲜事了，但是它和Docker for Windows不兼容往往很让人头大。通过查找资料，发现提供的解决方案大致有三种

 1. 先使用Vmware创建一台Linux虚拟机，在这台Linux虚拟机上再安装docker。
 2. 配置Vmware作为Docker for Windows的运行平台。
 3. 使用微软的Hyper-v来创建虚拟机。

对我而言，第一种不太优雅，第二种配置繁琐，第三种不会用。

直到我发现了vctl这个好东西。

# vctl 是什么？
> vctl 是一款捆绑在Vmware Workstation Pro 应用程序中的命令行实用程序，仅在 Windows 10 1809 或更高版本上受支持。如果 Workstation Pro 所在主机上的 Windows 操作系统低于 Windows 10 1809，则它不支持 vctl CLI。

简单来说它就是Vmware上的一个工具，可以用它来管理容器，使用命令基本上和docker一致，只需要把`docker <cmd>`换成`vctl <cmd>`就足够了。Docker for Windows？不需要。现在容器都交给vctl来管理了。

在使用vctl命令前，和启动docker一样，需要先启动vctl的守护进程。

```
vctl system start
```

当需要关闭守护进程时执行
```
vctl system stop
```

接下来就是和普通的docker命令一样了。

```
# 拉取镜像
vctl pull nginx

# 查看镜像
vctl images

# 启动容器
vctl --name some-nginx -d -p 8080:80 nginx

# 查看容器
vctl ps

# 进入容器
vctl exec -it <cid> bash 
```

更多使用信息可参考Vmware的官方文档 [使用vctl命令管理容器][1]


  [1]: https://docs.vmware.com/cn/VMware-Fusion/11/com.vmware.fusion.using.doc/GUID-78E7339F-7294-4F3E-9AD0-1E14C201FA40.html