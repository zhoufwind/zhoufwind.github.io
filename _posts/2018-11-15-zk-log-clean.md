---
layout: post
title: "Kafka/Zookeeper文件清理"
subtitle: "自己打的日志请记得回收！"
date: 2018-11-15 02:36PM
catalog: true
tags:
    - 运维
    - Kafka
    - Zookeeper
---

### 背景

虽然标题写了Kafka，但这篇文章重点不在Kafka，而是Zookeeper。

早上收到了kafka集群服务器磁盘空间不足的报警，料想最近日志量怎么增加的这么凶？8小时的轮询时间都撑爆了？多大的访问量？

### 问题

#### 提出疑问

由于Kafka服务器最占磁盘空间到文件即生产者生产的日志消息，随着生产者数量的增加，以及日志大小的变化，正常来说Kafka磁盘占用量是一个逐步增长的趋势。

我们知道Kafka是通过`log.retention.hours`这个参数来控制日志持久化时间的，时间越久，持久化时间越长，但越占磁盘空间。

一般来说产线并不会对持久化时间长短有强需求，所以先前遇到Kafka集群磁盘空间问题，第一反应是调整Kafka集群日志轮询时间。

我们从由最早的3天，改到1天，直到改到现在的8小时，突然发现无论改多小，似乎还是不够用，但查了下日志量，并没有明显增加，中间肯定有问题。

#### 发现线索

于是登录了集群中某台Kakfa服务器，看了下磁盘已使用量，再进Kafka数据目录，发现Kafka的使用量才占总使用量的1/3左右：

```
#df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        58G   23G   33G  41% /
tmpfs            16G   16K   16G   1% /dev/shm
/dev/sda1        97M   30M   62M  33% /boot
/dev/sda5       769G  605G  164G  79% /home

#cd /home/data/kafka-logs/
#du -sh
237G	.
```

于是对全盘做了扫描，发现了问题所在：

```
#du -h --max-depth=1
368G	./zookeeper
236G	./kafka-logs
0	./es-data
604G	.
```

没错，真正吃磁盘的不是Kafka，而是zk！然而先前根本就没有关注过zk文件使用情况…

### 缓解

发现了问题所在，解决起来就比较容易了，[网上][1]了解了下zk占用磁盘空间较大的文件主要包含两块内容：

- zookeeper snapshot（快照）：内存数据的快照；
- zookeeper log（日志）：类似mysql的binlog，将所有与修改数据相关的操作记录在log中；

> 正常运行过程中，ZK会不断地把快照数据和事务日志输出到这两个目录，并且如果没有人为操作的话，ZK自己是不会清理这些文件的，需要管理员来清理。

于是根据快照及日志所在目录（该集群指定zk地址为：`dataDir=/home/data/zookeeper`），写了个定时删除脚本，执行后磁盘空间报警恢复：

```bash
#!/bin/bash

# 0 3 * * * /home/scripts/cleanupLogsFolder_rh-zk.sh

LOG_DIR=/home/data/zookeeper/version-2/
TTL=7

if [ -d "$LOG_DIR" ];then  
  find $LOG_DIR -type f -mtime +"$TTL" -name "log.*" | xargs -i rm -f {}  
  find $LOG_DIR -type f -mtime +"$TTL" -name "snapshot.*" | xargs -i rm -f {}  
else  
  echo "Folder: '$LOG_DIR' isn't exist! exit . . ."; exit  
fi
```

若zk单独配置过文件目录的话，根据实际情况调整相应配置：

```
# vi /home/soft/zookeeper/conf/zoo.cfg
dataLogDir=/home/soft/zookeeper-3.4.8/logs
dataDir=/home/soft/zookeeper-3.4.8/zookeeper/data
```

### 后记

另外，zk自3.4.0版本开始支持日志及snapshort的自动删除，[官方][2]称可通过`autopurge.snapRetainCount`及`autopurge.purgeInterval`这两个参数进行配置：

> Automatic purging of the snapshots and corresponding transaction logs was introduced in version 3.4.0 and can be enabled via the following configuration parameters autopurge.snapRetainCount and autopurge.purgeInterval.

> **autopurge.snapRetainCount**  
> **New in 3.4.0:** When enabled, ZooKeeper auto purge feature retains the autopurge.snapRetainCount most recent snapshots and the corresponding transaction logs in the dataDir and dataLogDir respectively and deletes the rest. Defaults to 3. Minimum value is 3.

> **autopurge.purgeInterval**  
> **New in 3.4.0:** The time interval in hours for which the purge task has to be triggered. Set to a positive integer (1 and above) to enable the auto purging. Defaults to 0.

但有同学[指出][3]，当zk集群十分繁忙当情况下，删除操作可能会影响到zk集群性能，所以一般建议再访问低谷时，采用定时任务当方式，进行文件清理。

具体采用哪种方式，视具体情况而定。

[1]: http://blog.51cto.com/nileader/932156 "【ZooKeeper Notes 9】ZooKeepr日志清理"
[2]: https://zookeeper.apache.org/doc/r3.4.8/zookeeperAdmin.html "ZooKeeper Administrator's Guide - A Guide to Deployment and Administration"
[3]: http://www.cnblogs.com/yuyijq/p/3438829.html "Zookeeper-Zookeeper的配置"