---
layout: post
title: "Kibana4后台数据丢失后恢复"
date: 2019-03-07 04:57PM
catalog: true
tags:
    - 运维
    - ELK
---

### 背景

某机房老es集群下线，移除所有数据节点，保留1台`master node`服务器，原因是老es集群`kibana4`上仍保留有所有产线elk查询流量入口，暂时还不能下线。

![img](/img/in-post/post-190307-es-replica-remove/WechatIMG2733.jpeg)

### 问题

老es集群`data node`下线后，kibana4首页无法打开，浏览器f12显示链接出现503:
![img](/img/in-post/post-190307-es-replica-remove/WechatIMG2729.jpeg)

登录kibana服务所在机器，发现kibana进程未开启，手工后台开启后，5601端口仍然未出现，直到前台启动后，错误信息才输出到控制台上：
```
{"name":"Kibana","hostname":"es-node-01","pid":46228,"level":30,"msg":"Elasticsearch is still initializing the kibana index... Trying again in 2.5 second.","time":"2019-03-07T08:08:16.985Z","v":0}
```

[网上][1]搜了下，原来是`kibana4`缺失了`.kibana`这个数据索引，所以导致了`Kibana4`进程无法启动。这才意识到把数据节点下线后，kibana4的数据索引也丢了。

### 缓解

1. 立即将es `data node`的`es`服务再次启动，所有`.kibana`索引均从原先的`unassgin`状态，分别到各自所在的node上；

2. 修改`master node`配置文件，将其既充当`master node`，又充当`data node`；

3. 关闭es自置位功能，防止后面对分片手工置位后，es自动将其他分片置位以保持分片置位平均对功能；
```
PUT /_cluster/settings
{
 "transient": {
   "cluster.routing.allocation.enable": "none"
 }
}
```

4. 开始手工调整主分片置位，将所有`.kibana`索引置位至原master节点：
```
POST /_cluster/reroute
{
    "commands" : [
        {
            "move" : {
                "index" : ".kibaba", "shard" : 0,
                "from_node" : "elastic_inst8", "to_node" : "elastic_inst1"
            }
        }
    ]
}
```

5. 由于稍后只保留一个`data node`，也就意味着无法在`master/data node`上置主分片及副本，需要将副本关闭，接口可参照[官网][2]：
```
PUT /.kibana/_settings
{
  "number_of_replicas": 0
}
```
注：如果需要关闭所有索引的副本，可执行以下命令，但必须慎用，因为如果再要开启副本，对于索引量较大的索引，全部复制操作会消耗大量时间：
```
PUT /_settings
{
  "number_of_replicas": 0
}
```

6. 当前主分片应该已经置位到了`master/data node`，数据迁移完毕，可以安心关闭老`data node`了。
![img](/img/in-post/post-190307-es-replica-remove/WechatIMG2732.png)

确认置位成功后，再次开启自动置位功能：
```
PUT /_cluster/settings
{
 "transient": {
   "cluster.routing.allocation.enable": "all"
 }
}
```

### 后记

这已经是下老es集群后出现的第二例问题了，还好这个问题处理起来还算容易，但还是暴露了我们做变更操作不够谨慎的问题：  
  - 删除操作前必须确认所有数据是否已经备份（这边的数据不仅包含了应用程序数据，还包括了系统数据）

[1]: http://www.4wei.cn/archives/1002492 "ELK中无法启动kibana，解决“Elasticsearch is still initializing the kibana index... ”"
[2]: https://www.elastic.co/guide/en/elasticsearch/guide/master/replica-shards.html "Replica Shards"