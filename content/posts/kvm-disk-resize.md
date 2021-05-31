---
title: "KVM 虚拟机磁盘扩容"
categories: [ "虚拟化" ]
tags: [ "linux","kvm" ]
draft: false
slug: "kvm-disk-resize"
date: "2021-05-31T19:20:00+08:00"
---

### 一、镜像扩容

注意：需要先关闭虚拟机才能操作，`+`号前面有空格，后面没有空格。

```bash
qemu-img resize test.qcow2 +80G
```

原镜像磁盘大小20GB，扩容完成后可使用以下命令查看

```bash
qemu-img info test.qcow2
```

输出

```bash
image: test.qcow2
file format: qcow2
virtual size: 100G (107374182400 bytes)
disk size: 885M
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
```

### 二、Windows磁盘扩容

Windows磁盘扩容比较方便，进入 **计算机管理>磁盘管理** 找到新增的分区把它添加到需要的分区即可。

### 三、Linux磁盘扩容

启动虚拟机后，进入虚拟机控制台，使用`fdisk -l`命令查看磁盘信息。

```bash
Disk /dev/vda: 100 GiB, 107374182400 bytes, 209715200 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xe11f7f01

Device     Boot   Start      End  Sectors Size Id Type
/dev/vda1  *       2048  2099199  2097152   1G 83 Linux
/dev/vda2       2099200 41943039 39843840  19G 8e Linux LVM


Disk /dev/mapper/cl-root: 17 GiB, 18249416704 bytes, 35643392 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/cl-swap: 2 GiB, 2147483648 bytes, 4194304 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

可以看到这台虚拟机的磁盘大小已经有100GB了，但分区大小还是没有变化，只有初始大小20GB。

使用命令`fdisk /dev/vda`进行分区管理。

```bash
Welcome to fdisk (util-linux 2.32.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.
Command (m for help): n
```

输入 `n` 创建一个新分区并回车。

```bash
Partition type
   p   primary (2 primary, 0 extended, 2 free)
   e   extended (container for logical partitions)
Select (default p): p
```

提示选择分区，输入 `p`  选择主分区并回车。

```bash
Partition number (3,4, default 3):
```

提示选择分区编号，直接回车。

```bash
First sector (41943040-209715199, default 41943040): 
Last sector, +sectors or +size{K,M,G,T,P} (41943040-209715199, default 209715199): 
Created a new partition 3 of type 'Linux' and of size 80 GiB.
Command (m for help): t
```

输入`t`修改磁盘格式。

```
Partition number (1-3, default 3): 
Hex code (type L to list all codes): 8e
Changed type of partition 'Linux' to 'Linux LVM'.
Command (m for help): w
```

提示选择分区直接回车，提示选择十六进制编码，输入`8e`并回车，最后输入`w`保存并退出此流程。

再次输入`fdisk -l` 查看磁盘信息。

```bash
Disk /dev/vda: 100 GiB, 107374182400 bytes, 209715200 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xe11f7f01

Device     Boot    Start       End   Sectors Size Id Type
/dev/vda1  *        2048   2099199   2097152   1G 83 Linux
/dev/vda2        2099200  41943039  39843840  19G 8e Linux LVM
/dev/vda3       41943040 209715199 167772160  80G 8e Linux LVM


Disk /dev/mapper/cl-root: 17 GiB, 18249416704 bytes, 35643392 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/cl-swap: 2 GiB, 2147483648 bytes, 4194304 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

可以看到已经多了一个`/dev/vda3`分区，并且大小为80GB。

创建一个新的`pv`并添加到要扩容的`vg`中。

```
[root@localhost ~]# pvcreate /dev/vda3
  Physical volume "/dev/vda3" successfully created.
[root@localhost ~]# 
[root@localhost ~]# vgextend  cl /dev/vda3
  Volume group "cl" successfully extended
[root@localhost ~]# 
```

使用`vgdisplay`可以查看到已经扩容。

```bash
[root@localhost ~]# vgdisplay 
  --- Volume group ---
  VG Name               cl
  System ID             
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  4
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               98.99 GiB
  PE Size               4.00 MiB
  Total PE              25342
  Alloc PE / Size       4863 / <19.00 GiB
  Free  PE / Size       20479 / <80.00 GiB
  VG UUID               DHf6m6-x6YZ-S9ZT-VMTw-ylo5-NnLT-ozpnjR

```

使用 `df -TH` 查看文件信息。

```
[root@localhost ~]# df -TH
Filesystem          Type      Size  Used Avail Use% Mounted on
devtmpfs            devtmpfs   17G     0   17G   0% /dev
tmpfs               tmpfs      17G     0   17G   0% /dev/shm
tmpfs               tmpfs      17G  9.1M   17G   1% /run
tmpfs               tmpfs      17G     0   17G   0% /sys/fs/cgroup
/dev/mapper/cl-root xfs        19G  2.0G   17G  11% /
/dev/vda1           ext4      1.1G  135M  818M  15% /boot
tmpfs               tmpfs     3.4G     0  3.4G   0% /run/user/0
```

可以看出`/dev/mapper/cl-root` 挂载到了根目录，正是我们需要扩容的磁盘。使用以下命令进行扩容

```
lvresize -L +80G /dev/mapper/cl-root
```

大概率会出现

```
  Insufficient free space: 20480 extents needed, but only 20479 available
```

莫名其妙的被占用了一点点磁盘空间，所以需要修改一下扩容命令。

```
lvresize -L +79G /dev/mapper/cl-root
```

根据文件信息可以看出`/dev/mapper/cl-root` 的类型是 `xfs`，`xfs`类型的磁盘使用命令

```
xfs_growfs /dev/mapper/cl-root
```

其他类型使用

```
resize2fs /dev/mapper/cl-root
```

最后使用 `df -TH` 查看文件信息，可以看到已经扩容到了100GB。

```
[root@localhost ~]# df -TH
Filesystem          Type      Size  Used Avail Use% Mounted on
devtmpfs            devtmpfs   17G     0   17G   0% /dev
tmpfs               tmpfs      17G     0   17G   0% /dev/shm
tmpfs               tmpfs      17G  9.1M   17G   1% /run
tmpfs               tmpfs      17G     0   17G   0% /sys/fs/cgroup
/dev/mapper/cl-root xfs       104G  2.6G  101G   3% /
/dev/vda1           ext4      1.1G  135M  818M  15% /boot
tmpfs               tmpfs     3.4G     0  3.4G   0% /run/user/0
```