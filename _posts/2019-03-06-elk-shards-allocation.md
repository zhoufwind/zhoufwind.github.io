---
layout: post
title: "ES Data Node宕机后索引分片重平衡操作记录"
date: 2019-03-06 04:29PM
catalog: true
tags:
    - 运维
    - ELK
---

### 背景 & 问题

因某机房部署了新的es集群，老es集群服务器计划全部下架处理。

3月5日晚6点左右，首先从开始关闭es服务进程开始，关闭过程中意外将新es集群中某data node节点（下称节点a）的es进程关闭，随即导致新es集群集群状态发生变化：

1. 节点a被剔除新es集群，原先分布在该数据节点上的分片数据全部丢失；
2. 其他各data node根据分片属性，进行自调整，其中：  
    - 若现数据节点包含主分片数据，则随机复制一份副本至任意data node（除本节点外）；
    - 若现数据节点包含分片副本，副本自动置为主分片，并且随机复制一份副本至任意data node（除本节点外）。

操作失误后未及时发现，且未验证新老es集群状态，遂导致节点a一直处于下线状态，直到晚上收到监控报警：  
  - 晚8点07分收到新es集群索引出现延迟报警；
  - 晚8点50分左右收到某data node节点磁盘剩余空间小于25%报警。
  
但由于企业微信下班后自动屏蔽新消息，所以直到晚10点多，监控同学在微信群中人工反馈出来，这才开始介入排查。

排查很快就发现是关错es服务了，新集群正在进行分片重平衡，因索引数据重平衡会对索引性能造成影响，故出现索引延迟报警。

意识到这点后立刻将节点a的es服务开启，但此时已距离es服务下线4个多小时，节点a上的老索引数据已不可用，可视为节点a为一台全新的data node上线。

虽该操作自动触发了新es集群重平衡的操作，但遇到大索引，迁移一次分片耗时在几个小时以上，一晚上时间根本不足以完成所有data node索引的重平衡置位操作，所以在不经意间又犯了第二个错误：

**没有立即手工对索引进行重平衡**

该问题直接导致3月6日也就是今天，分片数据仍未完成平衡置位，自动重平衡仍在进行中，索引延迟仍然存在。

![img](/img/in-post/post-190306-elk-shards-allocation/WechatIMG2718.jpeg)

### 缓解

由于es自平衡只对分片数量进行筛选，不会针对索引大小进行筛选，所以必须人工介入手工重平衡，操作步骤如下：

1. 首先关闭es分片自动重平衡功能，接口文档参照[官网][1]：
```
PUT /_cluster/settings
{
    "transient": {
      "cluster.routing.allocation.enable": "none"
    }
}
```

2. 人肉查找出小索引，以便进行人工筛选：
![img](/img/in-post/post-190306-elk-shards-allocation/WechatIMG2720.jpeg)

3. 利用`Cerebro`整理出带人工迁移的索引、分片号、当前node节点以及目标node节点：
![img](/img/in-post/post-190306-elk-shards-allocation/WechatIMG2719.jpeg)

4. 利用`Kibana Dev Tools`手工对分片进行`reroute`，接口文档参照[官网][2]：
```
POST /_cluster/reroute
{
    "commands" : [
        {
            "move" : {
                "index" : "logstash-kafka-xxx-2019.03.04", "shard" : 1,
                "from_node" : "x.x.x.65", "to_node" : "x.x.x.62"
            }
        }
    ]
}
```

手工迁移完毕后，所有分片达到平衡状态，这样的话就可以保证明天新建的索引能够保持合理的分布状态了：
![img](/img/in-post/post-190306-elk-shards-allocation/WechatIMG2722.jpeg)

5. 操作完后千万不要忘记再把自动分片置位的功能开启，否则明天新索引就无法自动分片置位了：
```
PUT /_cluster/settings
{
    "transient": {
      "cluster.routing.allocation.enable": "all"
    }
}
```

### 后记

1. 做变更时必须仔细严谨，像关错服务下错机器这种低级错误特别致命！
2. 做完变更后需保持观察集群状态，验证新老服务是否正常运行，与目标状态是否保持一致；
3. 企业微信报警不能屏蔽；
4. 发现节点掉集群，应立即恢复es进程，在10分钟以内重新加入节点，索引不会进行迁移；
5. 发现节点掉集群，切重启es服务后老索引数据无法恢复的情况，应立即人工介入重平衡索引分片，避免第二天新索引集中置位在一个节点上；
6. 本次人工介入重平衡分片均为人肉手工操作，暂时还没发现有什么工具可以自动根据索引大小进行分片重平衡，由于迁移逻辑比较绕，且出现需求场景的频次也不是很高，这次就偷懒人肉操作了，后面有空会考虑写个脚本完成自匹配迁移。

[1]: https://www.elastic.co/guide/en/elasticsearch/reference/current/shards-allocation.html "Cluster Level Shard Allocation"
[2]: https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-reroute.html "Cluster Reroute"