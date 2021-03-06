---
title: "Linux 修改最大文件描述符"
categories: [ "Linux" ]
tags: [ "linux","sysctl","limit" ]
draft: false
slug: "linux-limit"
date: "2021-01-11 15:40:41"
---

```
echo "fs.file-max=655350" >>/etc/sysctl.conf
echo "* soft nofile 655350"  >> /etc/security/limits.conf
echo "* hard nofile 655350"  >> /etc/security/limits.conf
ulimit -n 655350
```