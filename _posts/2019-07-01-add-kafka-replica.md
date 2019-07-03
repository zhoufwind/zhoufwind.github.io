---
layout: post
title: "kafka副本扩容（笔记）"
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

### 补充

#### 格式问题

由于json文件内容是拷贝的，最后一个partition的结尾多了个逗号，提示以下报错：

```
Partitions reassignment failed due to Partition reassignment data file is empty
kafka.common.AdminCommandFailedException: Partition reassignment data file is empty
	at kafka.admin.ReassignPartitionsCommand$.parseAndValidate(ReassignPartitionsCommand.scala:183)
	at kafka.admin.ReassignPartitionsCommand$.executeAssignment(ReassignPartitionsCommand.scala:153)
	at kafka.admin.ReassignPartitionsCommand$.executeAssignment(ReassignPartitionsCommand.scala:149)
	at kafka.admin.ReassignPartitionsCommand$.main(ReassignPartitionsCommand.scala:46)
	at kafka.admin.ReassignPartitionsCommand.main(ReassignPartitionsCommand.scala)
```

把最后的逗号去掉后，扩容执行成功。

```
...
        },
        {
            "topic": "log4j_v1",
            "partition": 17,
            "replicas": [
                1,
                2
            ]
        }   <-- 删除这里多余的逗号
    ]
}
```

#### 执行log4j_v1副本扩容操作

扩容前parition及replicas的状态：

```
Topic:log4j_v1	PartitionCount:18	ReplicationFactor:1	Configs:
	Topic: log4j_v1	Partition: 0	Leader: 2	Replicas: 2	Isr: 2
	Topic: log4j_v1	Partition: 1	Leader: 0	Replicas: 0	Isr: 0
	Topic: log4j_v1	Partition: 2	Leader: 1	Replicas: 1	Isr: 1
	Topic: log4j_v1	Partition: 3	Leader: 1	Replicas: 1	Isr: 1
	Topic: log4j_v1	Partition: 4	Leader: 2	Replicas: 2	Isr: 2
	Topic: log4j_v1	Partition: 5	Leader: 0	Replicas: 0	Isr: 0
	Topic: log4j_v1	Partition: 6	Leader: 1	Replicas: 1	Isr: 1
	Topic: log4j_v1	Partition: 7	Leader: 2	Replicas: 2	Isr: 2
	Topic: log4j_v1	Partition: 8	Leader: 0	Replicas: 0	Isr: 0
	Topic: log4j_v1	Partition: 9	Leader: 2	Replicas: 2	Isr: 2
	Topic: log4j_v1	Partition: 10	Leader: 0	Replicas: 0	Isr: 0
	Topic: log4j_v1	Partition: 11	Leader: 1	Replicas: 1	Isr: 1
	Topic: log4j_v1	Partition: 12	Leader: 2	Replicas: 2	Isr: 2
	Topic: log4j_v1	Partition: 13	Leader: 0	Replicas: 0	Isr: 0
	Topic: log4j_v1	Partition: 14	Leader: 1	Replicas: 1	Isr: 1
	Topic: log4j_v1	Partition: 15	Leader: 2	Replicas: 2	Isr: 2
	Topic: log4j_v1	Partition: 16	Leader: 0	Replicas: 0	Isr: 0
	Topic: log4j_v1	Partition: 17	Leader: 1	Replicas: 1	Isr: 1
```

扩容后：

```
Topic:log4j_v1	PartitionCount:18	ReplicationFactor:2	Configs:
	Topic: log4j_v1	Partition: 0	Leader: 2	Replicas: 2,0	Isr: 2,0
	Topic: log4j_v1	Partition: 1	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 2	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: log4j_v1	Partition: 3	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: log4j_v1	Partition: 4	Leader: 2	Replicas: 2,0	Isr: 2,0
	Topic: log4j_v1	Partition: 5	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 6	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: log4j_v1	Partition: 7	Leader: 2	Replicas: 2,0	Isr: 2,0
	Topic: log4j_v1	Partition: 8	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 9	Leader: 2	Replicas: 2,0	Isr: 2,0
	Topic: log4j_v1	Partition: 10	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 11	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: log4j_v1	Partition: 12	Leader: 2	Replicas: 2,0	Isr: 2,0
	Topic: log4j_v1	Partition: 13	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 14	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: log4j_v1	Partition: 15	Leader: 2	Replicas: 2,0	Isr: 2,0
	Topic: log4j_v1	Partition: 16	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 17	Leader: 1	Replicas: 1,2	Isr: 1,2
```

#### 副本扩容后的高可用验证

在扩容完毕后，我们模拟一个broker宕机，验证kafka集群服务的高可用，手工kill broker 2的kafka进程，之后的分片及副本状态如下，broker 2脱离Isr(In-sync replica)配置，也即代表broker 2不可用：

```
Topic:log4j_v1	PartitionCount:18	ReplicationFactor:2	Configs:
	Topic: log4j_v1	Partition: 0	Leader: 0	Replicas: 2,0	Isr: 0
	Topic: log4j_v1	Partition: 1	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 2	Leader: 1	Replicas: 1,2	Isr: 1
	Topic: log4j_v1	Partition: 3	Leader: 1	Replicas: 1,2	Isr: 1
	Topic: log4j_v1	Partition: 4	Leader: 0	Replicas: 2,0	Isr: 0
	Topic: log4j_v1	Partition: 5	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 6	Leader: 1	Replicas: 1,2	Isr: 1
	Topic: log4j_v1	Partition: 7	Leader: 0	Replicas: 2,0	Isr: 0
	Topic: log4j_v1	Partition: 8	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 9	Leader: 0	Replicas: 2,0	Isr: 0
	Topic: log4j_v1	Partition: 10	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 11	Leader: 1	Replicas: 1,2	Isr: 1
	Topic: log4j_v1	Partition: 12	Leader: 0	Replicas: 2,0	Isr: 0
	Topic: log4j_v1	Partition: 13	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 14	Leader: 1	Replicas: 1,2	Isr: 1
	Topic: log4j_v1	Partition: 15	Leader: 0	Replicas: 2,0	Isr: 0
	Topic: log4j_v1	Partition: 16	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 17	Leader: 1	Replicas: 1,2	Isr: 1
```

我们同时升级了gohangout程序版本至最新版，这次模拟宕机操作并未重现当时整个es索引都受影响的现象，一切正常。

确认没问题后，我们重新启动了broker 2的kafka进程，此时分片及副本的状态如下，broker 2加回到Isr中，但作为备用节点，消费端消费数据不会再向broker 2拉取：

```
Topic:log4j_v1	PartitionCount:18	ReplicationFactor:2	Configs:
	Topic: log4j_v1	Partition: 0	Leader: 0	Replicas: 2,0	Isr: 0,2
	Topic: log4j_v1	Partition: 1	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 2	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: log4j_v1	Partition: 3	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: log4j_v1	Partition: 4	Leader: 0	Replicas: 2,0	Isr: 0,2
	Topic: log4j_v1	Partition: 5	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 6	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: log4j_v1	Partition: 7	Leader: 0	Replicas: 2,0	Isr: 0,2
	Topic: log4j_v1	Partition: 8	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 9	Leader: 0	Replicas: 2,0	Isr: 0,2
	Topic: log4j_v1	Partition: 10	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 11	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: log4j_v1	Partition: 12	Leader: 0	Replicas: 2,0	Isr: 0,2
	Topic: log4j_v1	Partition: 13	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 14	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: log4j_v1	Partition: 15	Leader: 0	Replicas: 2,0	Isr: 0,2
	Topic: log4j_v1	Partition: 16	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 17	Leader: 1	Replicas: 1,2	Isr: 1,2
```

#### 均衡分配

由于在宕机后，集群出现负载不均的问题，这边需要需要单独介入，均衡各分片所在broker，达到均衡负载的目的：

```
# ./bin/kafka-preferred-replica-election.sh --zookeeper zk1-base.xxxj.com:2181,zk2-base.xxxj.com:2181,zk3-base.xxxj.com:2181,zk4-base.xxxj.com:2181,zk5-base.xxxj.com:2181
Created preferred replica election path with 
{
	"version": 1,
	"partitions": [{
		"topic": "log4j_v1",
		"partition": 1
	}, {
		"topic": "log4j_v1",
		"partition": 6
	}, {
		"topic": "log4j_v1",
		"partition": 11
	}, {
		"topic": "log4j_v1",
		"partition": 16
	}, {
		"topic": "log4j_v1",
		"partition": 9
	}, {
		"topic": "log4j_v1",
		"partition": 0
	}, {
		"topic": "log4j_v1",
		"partition": 10
	}, {
		"topic": "log4j_v1",
		"partition": 12
	}, {
		"topic": "log4j_v1",
		"partition": 7
	}, {
		"topic": "log4j_v1",
		"partition": 4
	}, {
		"topic": "log4j_v1",
		"partition": 8
	}, {
		"topic": "log4j_v1",
		"partition": 13
	}, {
		"topic": "log4j_v1",
		"partition": 15
	}, {
		"topic": "log4j_v1",
		"partition": 3
	}, {
		"topic": "log4j_v1",
		"partition": 2
	}, {
		"topic": "log4j_v1",
		"partition": 14
	}, {
		"topic": "log4j_v1",
		"partition": 17
	}, {
		"topic": "log4j_v1",
		"partition": 5
	}, ...]
}
Successfully started preferred replica election for partitions Set([log4j_v1,5], [log4j_v1,10], [log4j_v1,2], [log4j_v1,14], [log4j_v1,13], [log4j_v1,0], [log4j_v1,15], [log4j_v1,4], [log4j_v1,12], [log4j_v1,17], [log4j_v1,11], [log4j_v1,9], [log4j_v1,1], [log4j_v1,3], [log4j_v1,6], [log4j_v1,8], [log4j_v1,7], [log4j_v1,16], ...)
```

操作完毕后，分片及副本状态如下：

```
Topic:log4j_v1	PartitionCount:18	ReplicationFactor:2	Configs:
	Topic: log4j_v1	Partition: 0	Leader: 2	Replicas: 2,0	Isr: 0,2
	Topic: log4j_v1	Partition: 1	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 2	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: log4j_v1	Partition: 3	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: log4j_v1	Partition: 4	Leader: 2	Replicas: 2,0	Isr: 0,2
	Topic: log4j_v1	Partition: 5	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 6	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: log4j_v1	Partition: 7	Leader: 2	Replicas: 2,0	Isr: 0,2
	Topic: log4j_v1	Partition: 8	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 9	Leader: 2	Replicas: 2,0	Isr: 0,2
	Topic: log4j_v1	Partition: 10	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 11	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: log4j_v1	Partition: 12	Leader: 2	Replicas: 2,0	Isr: 0,2
	Topic: log4j_v1	Partition: 13	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 14	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: log4j_v1	Partition: 15	Leader: 2	Replicas: 2,0	Isr: 0,2
	Topic: log4j_v1	Partition: 16	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: log4j_v1	Partition: 17	Leader: 1	Replicas: 1,2	Isr: 1,2
```

完成负载均衡。

### 参考文档

- [增加Kafka Topic的分区复本数(0.8.2.1)](http://blog.cheyo.net/272.html)
- [kafka topic增加replica报错解决](https://blog.csdn.net/huanggang028/article/details/49445569)
- [玩转zookeeper命令](https://www.cnblogs.com/sunsky303/p/8631432.html)
- [kafka how to delete topic with no leader](https://stackoverflow.com/questions/38139445/kafka-how-to-delete-topic-with-no-leader)
- [Kafka集群扩容遇到的问题](https://blog.51cto.com/jingfeng/1876505)
- [如何在Kafka中对Topic的leader进行均衡](https://blog.csdn.net/lizhitao/article/details/41441513)