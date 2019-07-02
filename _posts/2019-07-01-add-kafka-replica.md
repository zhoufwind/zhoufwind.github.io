---
layout: post
title: "kafka副本添加（笔记）"
date: 2019-07-01 05:37PM
catalog: true
tags:
    - 运维
    - Kafka
    - Zookeeper
---

### 背景

某机房kafka集群因其中一台节点服务器意外宕机，导致其中一个`broker`不可用，而`gohangout`在该情况下进程崩溃，无法继续消费，es索引出现延迟。

该问题记录在：[kafka集群中某broker宕机后gohangout panic](https://github.com/childe/gohangout/issues/41)

### 问题

gohangout进程崩溃存在问题，但kafka topic的配置也有问题，我们线上kafka topic都配置了0副本，如配置多副本的话，可能gohangout就不会有问题。

### 增加副本数

```
# ./bin/kafka-topics.sh --zookeeper zk1-base.xxxj.com:2181,zk2-base.xxxj.com:2181,zk3-base.xxxj.com:2181,zk4-base.xxxj.com:2181,zk5-base.xxxj.com:2181 --describe --topic iis
Topic:iis	PartitionCount:3	ReplicationFactor:1	Configs:
	Topic: iis	Partition: 0	Leader: 2	Replicas: 2	Isr: 2
	Topic: iis	Partition: 1	Leader: 0	Replicas: 0	Isr: 0
	Topic: iis	Partition: 2	Leader: 1	Replicas: 1	Isr: 1

# cat replication-iis.json 
{
    "version": 1, 
    "partitions": [
        {
            "topic": "iis", 
            "partition": 0, 
            "replicas": [
                2, 
                3 
            ]
        },
        {
            "topic": "iis", 
            "partition": 1, 
            "replicas": [
                0, 
                1
            ]
        },
        {
            "topic": "iis", 
            "partition": 2,
            "replicas": [
                1,
                2
            ]
        }
    ]
}
```

原本到这边，执行`execute`命令后增加副本的操作就结束了，但由于粗心，这边再配置p0的replica时写错了broker id，把`0`写成了`3`，发现后尝试修改，但发现怎么改也改不回去了...

```
# ./bin/kafka-reassign-partitions.sh --zookeeper zk1-base.xxxj.com:2181,zk2-base.xxxj.com:2181,zk3-base.xxxj.com:2181,zk4-base.xxxj.com:2181,zk5-base.xxxj.com:2181 --reassignment-json-file replication-iis.json --execute
Current partition replica assignment

{"version":1,"partitions":[{"topic":"iis","partition":2,"replicas":[1]},{"topic":"iis","partition":1,"replicas":[0]},{"topic":"iis","partition":0,"replicas":[2]}]}

Save this to use as the --reassignment-json-file option during rollback

# ./bin/kafka-reassign-partitions.sh --zookeeper zk1-base.xxxj.com:2181,zk2-base.xxxj.com:2181,zk3-base.xxxj.com:2181,zk4-base.xxxj.com:2181,zk5-base.xxxj.com:2181 --reassignment-json-file replication-iis.json --verify
Status of partition reassignment: 
Reassignment of partition [iis,0] is still in progress
Reassignment of partition [iis,1] completed successfully
Reassignment of partition [iis,2] completed successfully
```

修改`replication-iis.json`文件，尝试更新副本id，但提示无法执行`execute`操作，原因是zk还存着正在操作的信息：

```
# ./bin/kafka-reassign-partitions.sh --zookeeper zk1-base.xxxj.com:2181,zk2-base.xxxj.com:2181,zk3-base.xxxj.com:2181,zk4-base.xxxj.com:2181,zk5-base.xxxj.com:2181 --reassignment-json-file replication-iis.json --execute
There is an existing assignment running.
```

尝试手工删除zk中重分片的任务：

```
# ./bin/zkCli.sh
[zk: localhost:2181(CONNECTED) 2] get /admin/reassign_partitions
{"version":1,"partitions":[{"topic":"iis","partition":0,"replicas":[2,3]}]}
[zk: localhost:2181(CONNECTED) 3] rmr /admin/reassign_partitions
[zk: localhost:2181(CONNECTED) 4] get /admin/reassign_partitions
Node does not exist: /admin/reassign_partitions
```

再次执行，虽可执行`execute`，但并未生效，查看分片状态，发现zk中的replica信息未更新：

```
# ./bin/kafka-topics.sh --zookeeper zk1-base.xxxj.com:2181,zk2-base.xxxj.com:2181,zk3-base.xxxj.com:2181,zk4-base.xxxj.com:2181,zk5-base.xxxj.com:2181 --describe --topic iis
Topic:iis	PartitionCount:3	ReplicationFactor:2	Configs:
	Topic: iis	Partition: 0	Leader: 2	Replicas: 2,3	Isr: 2
	Topic: iis	Partition: 1	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: iis	Partition: 2	Leader: 1	Replicas: 1,2	Isr: 1,2
```

可以看到还是`Replicas: 2,3`，我们又尝试手工修改zk中的值：

```
[zk: localhost:2181(CONNECTED) 25] get /brokers/topics/iis
{"version":1,"partitions":{"2":[1,2],"1":[0,1],"0":[2,3]}}
[zk: localhost:2181(CONNECTED) 25] set /brokers/topics/iis {"version":1,"partitions":{"2":[1,2],"1":[0,1],"0":[2,0]}}
```

但再次执行`execute`还是无法生效...

```
# ./bin/kafka-topics.sh --zookeeper zk1-base.xxxj.com:2181,zk2-base.xxxj.com:2181,zk3-base.xxxj.com:2181,zk4-base.xxxj.com:2181,zk5-base.xxxj.com:2181 --describe --topic iis
Topic:iis	PartitionCount:3	ReplicationFactor:2	Configs:
	Topic: iis	Partition: 0	Leader: 2	Replicas: 2,3	Isr: 2
	Topic: iis	Partition: 1	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: iis	Partition: 2	Leader: 1	Replicas: 1,2	Isr: 1,2
```

此时内心将近崩溃，决定删了重建：

```
# ./bin/kafka-topics.sh --zookeeper zk1-base.xxxj.com:2181,zk2-base.xxxj.com:2181,zk3-base.xxxj.com:2181,zk4-base.xxxj.com:2181,zk5-base.xxxj.com:2181 --delete --topic iis
Topic iis is marked for deletion.
Note: This will have no impact if delete.topic.enable is not set to true.
```

再次创建，提示topic已存在，可查到topic信息：
```
# ./bin/kafka-topics.sh --zookeeper zk1-base.xxxj.com:2181,zk2-base.xxxj.com:2181,zk3-base.xxxj.com:2181,zk4-base.xxxj.com:2181,zk5-base.xxxj.com:2181 --create --topic iis --partitions 3 --replication-factor 2
Error while executing topic command : Topic 'iis' already exists.
ERROR org.apache.kafka.common.errors.TopicExistsException: Topic 'iis' already exists. 
(kafka.admin.TopicCommand$)

# ./bin/kafka-topics.sh --zookeeper zk1-base.xxxj.com:2181,zk2-base.xxxj.com:2181,zk3-base.xxxj.com:2181,zk4-base.xxxj.com:2181,zk5-base.xxxj.com:2181 --describe --topic iis
Topic:iis	PartitionCount:3	ReplicationFactor:2	Configs:
	Topic: iis	Partition: 0	Leader: -1	Replicas: 2,0	Isr: 
	Topic: iis	Partition: 1	Leader: -1	Replicas: 0,1	Isr: 
	Topic: iis	Partition: 2	Leader: -1	Replicas: 1,2	Isr: 
```

zk中删除value：
```
[zk: localhost:2181(CONNECTED) 26] rmr /brokers/topics/iis
[zk: localhost:2181(CONNECTED) 27] rmr /admin/reassign_partitions
```

再次创建：
```
# ./bin/kafka-topics.sh --zookeeper zk1-base.xxxj.com:2181,zk2-base.xxxj.com:2181,zk3-base.xxxj.com:2181,zk4-base.xxxj.com:2181,zk5-base.xxxj.com:2181 --create --topic iis --partitions 3 --replication-factor 2
Created topic "iis".
```

这次提示leader: none

```
# ./bin/kafka-topics.sh --zookeeper zk1-base.xxxj.com:2181,zk2-base.xxxj.com:2181,zk3-base.xxxj.com:2181,zk4-base.xxxj.com:2181,zk5-base.xxxj.com:2181 --describe --topic iis
Topic:iis	PartitionCount:3	ReplicationFactor:2	Configs:
	Topic: iis	Partition: 0	Leader: none	Replicas: 2,0	Isr: 
	Topic: iis	Partition: 1	Leader: none	Replicas: 0,1	Isr: 
	Topic: iis	Partition: 2	Leader: none	Replicas: 1,2	Isr: 
```

这时突然发现我的iis分片数据用的不是这个kafka集群，而是用的其他kafka集群，所以再怎么改新数据也不会进来。

好在折腾了一番没影响线上，下次操作还是要细心。

至于增加了副本后，broker宕机gohangout是否能够稳定运行，待测试后再做分析。