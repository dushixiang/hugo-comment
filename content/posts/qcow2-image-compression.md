---
title: "压缩qcow2 镜像文件"
categories: [ "kvm" ]
tags: [ "qcow2" ]
draft: false
slug: "qcow2-image-compression"
date: "2020-12-31 00:00:00"
---

# 压缩qcow2

首先，需要对虚拟机剩余空间进行写零操作：

```
dd if=/dev/zero of=/zero.dat
```

删除 zero.dat：

```
rm /zero.dat
```

关闭虚拟机，执行`qemu-img`的`convert`命令进行转换：

```
qemu-img convert -c -O qcow2 /path/old.img.qcow2 /path/new.img.qcow2
```