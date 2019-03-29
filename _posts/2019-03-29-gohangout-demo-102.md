---
layout: post
title: "gohangout运行情况阶段汇总"
date: 2019-03-29 03:00PM
catalog: true
tags:
    - 运维
    - ELK
    - Go
---

### 背景

自gohangout部署至生产环境，到今天已经过去了半个月时间了，期间运行的情况如何？遇到了些什么问题？在这边做个简单的小结。

### 运行情况

#### 进程意外中断

遇到gohangout进程意外中断问题，后确定是gohangout程序版本的问题。自行pull最新代码并build后再无类似问题出现，进程已稳定运行2周无任何异常。

```
panic: close of closed channel

goroutine 173797209 [running]:
github.com/childe/healer.(*Broker).requestFetchStreamingly.func1(0xc037639f98, 0xc016633560)
	/usr/local/lib/go/src/github.com/childe/healer/broker.go:426 +0x4d
github.com/childe/healer.(*Broker).requestFetchStreamingly(0xc000254de0, 0xc061ab6e40, 0xc016633560, 0x9f10a0, 0xc0491702d0)
	/usr/local/lib/go/src/github.com/childe/healer/broker.go:443 +0x141
github.com/childe/healer.(*SimpleConsumer).Consume.func2.2(0xc0000a0960, 0xc061ab6e40, 0xc016633560)
	/usr/local/lib/go/src/github.com/childe/healer/simple_consumer.go:324 +0x47
created by github.com/childe/healer.(*SimpleConsumer).Consume.func2
	/usr/local/lib/go/src/github.com/childe/healer/simple_consumer.go:323 +0x80f
```

#### 字段替换功能实现

提出了字段替换的需求，先用匹配赋值的方式实现，后增加程序功能，得到实现：

原logstash替换规则配置：  

```
filter {
  if [type] =~ "^hj-*" {
    mutate {
      gsub => [
        "s-response_time", "-", "0.000",
        "response_time", "-", "0.000"
      ]
    }
  }
}
```

gohangout临时过渡方案：  

```
filters:
    - Add:
        if:
            - 'EQ(type,"hj-apigw")'
            - 'EQ(s-response_time,"-")'
        fields:
            s-response_time: "0.000"
    - Add:
        if:
            - 'EQ(type,"hj-apigw-ssl")'
            - 'EQ(response_time,"-")'
        fields:
            response_time: "0.000"
```

gohangout新版本已支持替换功能：  

```
filters:
    - Replace:
        if:
            - 'EQ(type,"hj-apigw")'
        fields:
            s-response_time: ['-', '0.000']
    - Replace:
        if:
            - 'EQ(type,"hj-apigw-ssl")'
        fields:
            response_time: ['-', '0.000']
```

#### 针对log4j日志格式开发新的分割方式

我司java日志采用了标准log4j日志格式，并在其上做调整，通过针对`[]`进行`split`操作，替代正则匹配分割，大大减少了cpu利用率，增加了程序运行性能：

原正则匹配分割方式：  

```
filters:
    - Grok:
        if:
            - 'HasPrefix(type,hj-)'
        src: message
        match:
            - '^\[(?P<level>\S+)\] \[(?P<logtime_format>.*?)\,(?P<logtime_ms>.*?)\] \[(?P<logger>.*?)\] \[(?P<threadId>.*?)\] \[(?P<class>.*?)\] \[(?P<requestId>.*?)\] \[(?P<srv_ip>.*?)\] (?ms)(?P<msgRawData>.*)$'
        remove_fields: ['message']
        add_fields:
            grok_result: 'ok'
```

现利用`[]`进行分割方式：  

```
filters:
    - Split:
        if:
            - 'HasPrefix(type,hj-)'
        src: message
        sep: '] '
        trim: ']['
        maxSplit: 8
        fields: ['level', 'logtime', 'logger', 'threadId', 'class', 'requestId', 'srv_ip', 'msgRawData']
        remove_fields: ['message']
    - Split:
        if:
            - 'HasPrefix(type,hj-)'
        src: logtime
        sep: ","
        fields: ['logtime_format', 'logtime_ms']
        remove_fields: ["logtime"]
```

调整前cpu平均空闲率为`50.19%`，调整后为`60.74%`，cpu时间直接空出`10.55%`！

![img](/img/in-post/post-190329-gohangout-102/WechatIMG2855.jpeg)

#### 引用grok patterns作为分割规则

自己写正则匹配规则是比较痛苦的一件事情，同时写的不好还可能影响匹配效率，突然发现logstash的grok规则也可应用到gohangout配置，事半功倍！

nginx error日志采用grok规则进行匹配的示例：  

```
    - Grok:
        if:
            - 'EQ(type,"openresty-error")'
        src: message
        match:
            - '^(?P<logtime>%{YEAR}/%{MONTHNUM}/%{MONTHDAY} %{TIME}) \[(?P<severity>%{LOGLEVEL})\] (?P<pid>%{NUMBER})#%{NUMBER}: (?P<errormessage>%{GREEDYDATA})'
        ignore_blank: true
        remove_fields: ['message']
        add_fields:
            grok_result: 'ok'
        pattern_paths:
            #- 'https://raw.githubusercontent.com/vjeantet/grok/master/patterns/grok-patterns'
            - '/home/soft/gohangout/grok-patterns'
```

注意`grok-patterns`文件必须下载到本地进行读取，否则从网络拉取可能出现调用失败的情况：

![img](/img/in-post/post-190329-gohangout-102/WechatIMG2856.jpeg)

#### gohangout读取kafka offset的问题

有时为测试需求，需要使用同一个group.id反复读取kafka中数据的情况，这个时候即便配置了`from.beginning: true`，但启动gohangout后，group的offset仍然在`latest`，这样的话测试的话就比较麻烦了，必须用kafka的脚本修改offset：

```
# ./bin/kafka-consumer-groups.sh --bootstrap-server x.x.x.x:9092 --group gohangout-bi_event-qa-topic.test --describe
# ./bin/kafka-consumer-groups.sh --bootstrap-server x.x.x.x:9092 --group gohangout-bi_event-qa-topic.test --reset-offsets --topic event --to-earliest --execute
```

后经确认，只有在使用全新的group.id并配置了`from.beginning: true`，才会从最早开始读，否则全部以上次读到的offset开始传输。

#### gohangout与logstash数据量不一致？

在查询某天索引数据量时，发现logstash的索引量比gohangout多，是logstash数据有重呢？还是gohangout丢数据了呢？我们对其进行了排查。

![img](/img/in-post/post-190329-gohangout-102/WechatIMG2857.png)

减小查询范围，我们定位到了问题点，logstash下有2条相同的日志，而gohangout只有1条：

logstash的索引包含2条日志：  
![img](/img/in-post/post-190329-gohangout-102/WechatIMG2859.png)

而gohangout的索引只有1条日志：
![img](/img/in-post/post-190329-gohangout-102/WechatIMG2858.png)

我们现在还是无法判断究竟是logstash重，还是gohangout丢，只有上服务器查看服务器原日志，才能下结论。

通过回看原日志，我们发现确实只有一条，是logstash重复读了，重复的原因可能是kafka的rebalance造成的。

#### gohangout取不到kafka数据？查下java状态！

在使用gohangout时突然遇到以下报错：  

```
E0329 13:19:56.433807  140801 simple_consumer.go:326] fetch error:read tcp 192.168.36.240:20042->192.168.38.112:9092: i/o timeout
E0329 13:19:56.433864  140801 fetch_response.go:226] could read enough bytes(4) to get fetchresponse length. read 0 bytes
E0329 13:19:56.434213  140801 simple_consumer.go:326] fetch error:read tcp 192.168.36.240:20044->192.168.38.112:9092: i/o timeout
E0329 13:19:56.434256  140801 fetch_response.go:226] could read enough bytes(4) to get fetchresponse length. read 0 bytes
```

后查看kafka日志，发现以下异常，发现kafka oom...  

```
[2019-03-29 15:08:01,365] ERROR [KafkaApi-1] Error when handling request {replica_id=-1,max_wait_time=100,min_bytes=1,topics=[{topic=event,partitions=[{partition=1,fetch_offset=0,max_bytes=10485760}]}]} (kafka.server.KafkaApis)
java.lang.OutOfMemoryError: Java heap space
        at java.nio.HeapByteBuffer.<init>(HeapByteBuffer.java:57)
        at java.nio.ByteBuffer.allocate(ByteBuffer.java:335)
        at org.apache.kafka.common.record.AbstractRecords.downConvert(AbstractRecords.java:101)
        at org.apache.kafka.common.record.FileRecords.downConvert(FileRecords.java:242)
        at kafka.server.KafkaApis$$anonfun$kafka$server$KafkaApis$$convertedPartitionData$1$1$$anonfun$apply$4.apply(KafkaApis.scala:550)
        at kafka.server.KafkaApis$$anonfun$kafka$server$KafkaApis$$convertedPartitionData$1$1$$anonfun$apply$4.apply(KafkaApis.scala:548)
        at scala.Option.map(Option.scala:146)
        at kafka.server.KafkaApis$$anonfun$kafka$server$KafkaApis$$convertedPartitionData$1$1.apply(KafkaApis.scala:548)
        at kafka.server.KafkaApis$$anonfun$kafka$server$KafkaApis$$convertedPartitionData$1$1.apply(KafkaApis.scala:538)
        at scala.Option.flatMap(Option.scala:171)
        at kafka.server.KafkaApis.kafka$server$KafkaApis$$convertedPartitionData$1(KafkaApis.scala:538)
        at kafka.server.KafkaApis$$anonfun$kafka$server$KafkaApis$$createResponse$2$1.apply(KafkaApis.scala:579)
        at kafka.server.KafkaApis$$anonfun$kafka$server$KafkaApis$$createResponse$2$1.apply(KafkaApis.scala:575)
        at scala.collection.Iterator$class.foreach(Iterator.scala:891)
        at scala.collection.AbstractIterator.foreach(Iterator.scala:1334)
        at scala.collection.IterableLike$class.foreach(IterableLike.scala:72)
        at scala.collection.AbstractIterable.foreach(Iterable.scala:54)
        at kafka.server.KafkaApis.kafka$server$KafkaApis$$createResponse$2(KafkaApis.scala:575)
        at kafka.server.KafkaApis$$anonfun$kafka$server$KafkaApis$$fetchResponseCallback$1$2.apply(KafkaApis.scala:596)
        at kafka.server.KafkaApis$$anonfun$kafka$server$KafkaApis$$fetchResponseCallback$1$2.apply(KafkaApis.scala:596)
        at kafka.server.KafkaApis$$anonfun$sendResponseMaybeThrottle$1.apply$mcVI$sp(KafkaApis.scala:2223)
        at kafka.server.ClientRequestQuotaManager.maybeRecordAndThrottle(ClientRequestQuotaManager.scala:54)
        at kafka.server.KafkaApis.sendResponseMaybeThrottle(KafkaApis.scala:2222)
        at kafka.server.KafkaApis.kafka$server$KafkaApis$$fetchResponseCallback$1(KafkaApis.scala:596)
        at kafka.server.KafkaApis$$anonfun$kafka$server$KafkaApis$$processResponseCallback$1$1.apply$mcVI$sp(KafkaApis.scala:614)
        at kafka.server.ClientQuotaManager.maybeRecordAndThrottle(ClientQuotaManager.scala:176)
        at kafka.server.KafkaApis.kafka$server$KafkaApis$$processResponseCallback$1(KafkaApis.scala:613)
        at kafka.server.KafkaApis$$anonfun$handleFetchRequest$4.apply(KafkaApis.scala:630)
        at kafka.server.KafkaApis$$anonfun$handleFetchRequest$4.apply(KafkaApis.scala:630)
        at kafka.server.ReplicaManager.fetchMessages(ReplicaManager.scala:835)
        at kafka.server.KafkaApis.handleFetchRequest(KafkaApis.scala:622)
        at kafka.server.KafkaApis.handle(KafkaApis.scala:105)
```

通过进程查看kafka jvm参数，`-Xmx1G -Xms1G`实在太小，调整配置后恢复。

```
export KAFKA_HEAP_OPTS="-Xmx10G -Xms10G"
```
