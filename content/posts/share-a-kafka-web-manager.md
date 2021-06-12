---
title: "分享一款非常好用的kafka可视化web管理工具"
categories: [ "kafka" ]
tags: [ "kafka" ]
draft: false
slug: "share-a-kafka-web-manager"
date: "2021-06-13T00:00:00+08:00"
---


使用过`kafka`的小伙伴应该都知道`kafka`本身是没有管理界面的，所有操作都需要手动执行命令来完成。但有些命令又多又长，如果没有做笔记，别说是新手，就连老手也不一定能记得住，每次想要使用的时候都要上网搜索一下。有些崇尚geek精神的人或许觉得命令行才是真爱，但使用一款好用的可视化管理工具真的可以极大的提升效率。

今天给大家介绍的这款工具叫做`kafka-map`，是我针对日常工作中高频使用的场景开发的，使用了这款工具之后就不必费心费力的去查资料某个命令要怎么写，就像是：“给编程插上翅膀，给kafka装上导航”。

## kafka-map 介绍

`kafka map`是使用`Java11`和`React`开发的一款`kafka`可视化工具。

目前支持的功能有：

- 多集群管理
- 集群状态监控（分区数量、副本数量、存储大小、offset）
- 主题创建、删除、扩容（删除需配置delete.topic.enable = true）
- broker状态监控
- 消费者组查看、删除
- 重置offset
- 消息查询（支持String和json方式展示）
- 发送消息（支持向指定的topic和partition发送字符串消息）

## 功能截图

### 添加集群

![添加集群](https://gitee.com/dushixiang/kafka-map/raw/master/screenshot/import-cluster.png)

### 集群管理

![集群管理](https://gitee.com/dushixiang/kafka-map/raw/master/screenshot/clusters.png)

### broker

![broker](https://gitee.com/dushixiang/kafka-map/raw/master/screenshot/brokers.png)

### 主题管理

![主题管理](https://gitee.com/dushixiang/kafka-map/raw/master/screenshot/topics.png)

### 消费组

![消费组](https://gitee.com/dushixiang/kafka-map/raw/master/screenshot/consumers.png)

### 查看消费组已订阅主题

![消费组详情](https://gitee.com/dushixiang/kafka-map/raw/master/screenshot/consumer-subscription.png)

### topic详情——分区

![topic详情——分区](https://gitee.com/dushixiang/kafka-map/raw/master/screenshot/topic-info-partition.png)

### topic详情——broker

![topic详情——broker](https://gitee.com/dushixiang/kafka-map/raw/master/screenshot/topic-info-broker.png)

### topic详情——消费组

![topic详情——消费组](https://gitee.com/dushixiang/kafka-map/raw/master/screenshot/topic-info-consumer.png)

### topic详情——消费组重置offset

![topic详情——消费组重置offset](https://gitee.com/dushixiang/kafka-map/raw/master/screenshot/topic-info-consumer-reset-offset.png)

### topic详情——配置信息

![topic详情——配置信息](https://gitee.com/dushixiang/kafka-map/raw/master/screenshot/topic-info-config.png)

### 生产消息

![消费消息](https://gitee.com/dushixiang/kafka-map/raw/master/screenshot/producer-message.png)

### 消费消息

![消费消息](https://gitee.com/dushixiang/kafka-map/raw/master/screenshot/consumer-message.png)

## docker 方式安装

一行命令即可完成安装

```shell
docker run -d \
    -p 8080:8080 \
    -v /opt/kafka-map/data:/usr/local/kafka-map/data \
    -e DEFAULT_USERNAME=admin
    -e DEFAULT_PASSWORD=admin
    --name kafka-map \
    --restart always dushixiang/kafka-map:latest
```

更多安装方式以及相信信息可查看: https://github.com/dushixiang/kafka-map

欢迎star和分享给其他小伙伴。