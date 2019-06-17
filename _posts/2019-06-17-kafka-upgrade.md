---
layout: post
title: "Kafka版本升级记录（v1.1.1 -> v2.2.1）"
date: 2019-06-17 05:57PM
catalog: true
tags:
    - 运维
    - Kafka
---

### 背景

从`1.1.x`升级到`2.2.x`，跨越了1个大版本，主要提升了SSL、安全验证、内存使用优化、程序健壮性、新版本java支持、API改进等功能。

升级步骤按照官方文档即可，值得注意的是，从`1.1.x`升级至`2.2.x`，无需指定消息格式配置，也即：`log.message.format.version`这项，只有低于`0.11.0.x`版本才需单独制定。

### 升级步骤

首先下载新版kafka程序，修改配置文件，指定`inter.broker.protocol.version=1.1.1`，其他配置保持不变，重启kafka，重启zk（如果用的是本地zk的话），一台重启完后接着下一台，直至集群节点全部重启完毕。

第一轮重启完后，确认没问题就可以修改配置：`inter.broker.protocol.version=2.2.1`，接着第二轮重启操作，此步只需重启kafka，不用重启zk（如果用的是本地zk的话）。

注意第一轮重启后还可以进行回滚操作，第二轮重启后无法回退至先前版本。

第二轮重启完毕后，升级结束，可明显发现内存占用率是原先老版本时的一半，确实得到了很好的优化。

### 参考文档

[Apache Kafka Download](https://kafka.apache.org/downloads)  
[Apache Kafka Document - upgrade 2.2.0](https://kafka.apache.org/documentation/#upgrade_2_2_0)