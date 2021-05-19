---
title: "使用libvirt-java采集KVM虚拟机状态信息"
categories: [ "虚拟化" ]
tags: [ "Java","libvirt" ]
draft: false
slug: "collect-vm-stats-by-libvirt-java"
date: "2021-05-19T20:18:20+08:00"
---

虚拟化开发相较于普通开发是一个冷门的方向，大多数是使用Python开发，其中使用Java来做虚拟化的少之又少，资料更是少的可怜，为了实现需求我也是踩了不少坑，今天就为大家分享一下如何使用 `libvirt-java` 来采集KVM虚拟机的资源使用信息。

### CPU使用率

`libvirt`并没有直接提供获取虚拟机CPU使用率的接口，需要我们自己来计算，网上分享的代码或者公式五花八门，大部分都是错误的，经过我的测试，找到了一个相对准确的计算公式。

```latex
cpu_usage = (cpu_time_now - cpu_time_t_second_ago) * 100 / (t * vCpus * 10^9)
```

Java代码如下

```java
// t秒前的CPU时间
long c1 = domain.getInfo().cpuTime;
Thread.sleep(1000);
// 当前CPU时间
long c2 = domain.getInfo().cpuTime;

// 虚拟CPU数量
int vCpus = domain.getMaxVcpus();
// t 为1秒
Double cpuUsage = 100 * (c2 - c1) / (1 * vCpus * Math.pow(10, 9));
log.debug("虚拟机[{}]CPU使用率为: {}", uuid, cpuUsage);
```

### 内存使用率

不要使用`domain.getInfo()`返回的 `memory`字段，虽然它注释写的是`the memory in KBytes used by the domain`，但它的意思真的不是虚拟机内部进程已使用的内存大小，而是从宿主机器的角度来看分配给这个虚拟机的内存它使用了多少，如果没有特殊配置，它会和`maxMem`字段的值是相同的。

正确做法是使用`domain.memoryStats(10)`来获取，那为什么参数要输入一个`10`呢？这是因为`10`代表的是要返回的信息数量，经过我手动执行`virsh dommemstat uuid` 测试发现有10个参数返回，所以需要填入`10`。另外命令返回的`unused` 字段值与数组中`tag=8`的数据一致，最终我们获取到了未使用的内存大小，计算内存使用率更是轻轻松松。

Java代码如下

```java
MemoryStatistic[] memoryStatistics = domain.memoryStats(10);
Optional<MemoryStatistic> first = Arrays.stream(memoryStatistics).filter(x -> x.getTag() == 8).findFirst();
if (first.isPresent()) {
  MemoryStatistic memoryStatistic = first.get();
  long unusedMemory = memoryStatistic.getValue();
  long maxMemory = domain.getMaxMemory();
  double memoryUsage = (maxMemory - unusedMemory) * 100.0 / maxMemory;
  log.debug("虚拟机[{}]内存使用率为: {}", uuid, memoryUsage);
}
```

### 网卡数据包信息

同样`libvirt`并没有提供获取虚拟机网卡的接口，因此需要获取虚拟机的xml文件来查询。

此处没有什么坑，解析`xml`是使用了`html`解析库`Jsoup`，`xml`算是`html`的亲戚吧，比`html`书写严格很多，解析数据更为方便。

获取到网卡名称之后再获取统计信息，可以获取的数据有：

| 字段       | 含义               |
| ---------- | ------------------ |
| rx_bytes   | 接收数据包大小     |
| rx_packets | 接收数据包数量     |
| rx_errs    | 接收错误数据包数量 |
| rx_drop    | 接收丢弃数据包数量 |
| tx_bytes   | 发送数据包大小     |
| tx_packets | 发送数据包数量     |
| tx_errs    | 发送错误数据包数量 |
| tx_drop    | 发送丢弃数据包数量 |

Java代码如下

```java
String xmlDesc = domain.getXMLDesc(0);
Document document = Jsoup.parse(xmlDesc);

// 网卡
Elements interfaces = document.getElementsByTag("devices").get(0).getElementsByTag("interface");
for (Element inter : interfaces) {
  String interName = inter.getElementsByTag("target").get(0).attr("dev");
  DomainInterfaceStats domainInterfaceStats = domain.interfaceStats(interName);
  log.debug("dev {} stats {}", interName, Json.toJsonString(domainInterfaceStats));
}
```

### 磁盘IO信息

和网卡同样，`libvirt`也没有提供获取虚拟机磁盘的接口，还是需要获取虚拟机的xml文件来查询，获取到磁盘名称之后再获取统计信息，可以获取的数据有：

| 字段     | 含义           |
| -------- | -------------- |
| rd_req   | 读取请求总数   |
| rd_bytes | 读取的数据大小 |
| wr_req   | 写入请求总数   |
| wr_bytes | 写入的数据大小 |
| errs     | 失败次数       |

Java代码如下

```java
String xmlDesc = domain.getXMLDesc(0);
Document document = Jsoup.parse(xmlDesc);
// 磁盘
Elements disks = document.getElementsByTag("devices").get(0).getElementsByTag("disk");
for (Element disk : disks) {
  String dev = disk.getElementsByTag("target").get(0).attr("dev");
  DomainBlockStats domainBlockStats = domain.blockStats(dev);
  log.debug("dev {} stats {}", dev, Json.toJsonString(domainBlockStats));
}
```

至此采集虚拟机状态信息算是告一段落，学的越多才发现不会的越多...
