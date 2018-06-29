---
layout: post
title: "修复数据格式冲突导致ES索引失败的问题"
subtitle: "曲线救国，统一不了那就干脆删了吧！"
date: 2018-06-29 11:46:00 +0800
catalog: ture
tags:
    - 运维
    - ELK
---
### 背景
开发人员由于业务需要，在日志中通过`log_time`字段记录用户真实发起请求时间，然而处于某些原因，新老日志格式不同步，字段格式存在不一致的情况，包括：
- 时间戳：1530241094655
- TIMESTAMP_ISO8601：2018-06-29T02:46:34.151Z

Kibana早已显示有冲突了：
![img](/img/in-post/post-NumberFormatException/post-NumFormEx-form-conflict.jpeg)
![img](/img/in-post/post-NumberFormatException/post-NumFormEx-form-conflict-2.jpeg)

### 问题
早上接到用户反馈，Kibana打开异常，ES挂了：

![img](/img/in-post/post-NumberFormatException/post-NumFormEx-es-down.jpeg)

马上查了下9200端口，能够正常返回，说明当前应该还行，不算太遭。

马上再查ES索引数，确实出现了异常，集群索引数跌了下来：

![img](/img/in-post/post-NumberFormatException/post-NumFormEx-es-index-fail.jpeg)

看下ES节点CPU情况，果然io wait有突增：

![img](/img/in-post/post-NumberFormatException/post-NumFormEx-high-io-wait.jpeg)

第一反应怀疑磁盘是不是有问题，但监控反馈并无磁盘坏道报警，IDC代维也查了下几台ES服务器，硬盘灯均正常，看来不是硬件问题。

搜ES日志，确实在抛大量异常，如：
```
[2018-06-29T10:40:41,037][DEBUG][o.e.a.b.TransportShardBulkAction] [xxx] [logstash-kafka-node_v1-2018.06.29][3] failed to execute bulk item (index) index {[logstash-kafka-node_v1-2018.06.29][node-web][AWRJagAyxJcsoTmKk_hX], source[{"referer":"http://www.hjenglish.com/sijichengji/cet4chengjigongbushijian/","s_ip":"xxx","useragent":"Mozilla/5.0 (Windows NT 5.1; rv:8.0.1) Gecko/20100101 Firefox/8.0.1","source":"/logpath/logstash_web.log-2018-06-28","c_bytes":0,"type":"node-web","path":"/sijichengji/cet4chengjigongbushijian/","beat":{"hostname":"xxx","name":"xxx","version":"5.5.0"},"host":"www.hjenglish.com","@version":"1","method":"GET","offset":93453915,"input_type":"log","proj":"Node.EN","c_ip":"180.149.143.140","time_third_api":0,"log_time":"2018-06-29T02:29:05.549Z","time_taken":48,"@timestamp":"2018-06-29T02:29:05.774Z","user_id":0,"true":"","s_bytes":41287,"s_name":"www.hjenglish.com","request_id":"dd4df638bca5494bb018dd5e80ac53f2","query_string":"{}","status":200}]}
org.elasticsearch.index.mapper.MapperParsingException: failed to parse [log_time]
	at org.elasticsearch.index.mapper.FieldMapper.parse(FieldMapper.java:298) ~[elasticsearch-5.2.2.jar:5.2.2]
	...
Caused by: java.lang.NumberFormatException: For input string: "2018-06-29T02:29:05.549Z"
	at java.lang.NumberFormatException.forInputString(NumberFormatException.java:65) ~[?:1.8.0_131]
	at java.lang.Long.parseLong(Long.java:589) ~[?:1.8.0_131]
	...
```

这时可以确认是因为`NumberFormatException`引起的问题，之前也因为这个原因挂过几次，但不多，而且每次都是过了段时间自己恢复了。

但这次既然又出事了，还是想办法fix掉吧，避免下次再次发生。

### 缓解
事情本因开发日志格式不一致所导致，所以还是想着让开发去进行调整，可供参考的策略如下：
- 日志格式标准统一起来，解决问题
- Node日志分离出来，单独搞套集群，保证大集群不受影响，减少影响范围
- Node日志从集群上下掉

但开发在群里一句话都没回，暂且不说什么原因不回，但相信推动难度会相当大，很有可能的结果是没有结果。

所以这边开始想，如何能够在现有日志不做调整的情况下，保证日志不冲突？至少不影响到ES集群稳定性？

到这边，突然想到个办法：直接把`log_time`这个字段先丢弃不采集，这样入ES应该不会再报NumberFormatException了。

于是立即调整了下logstash indexer的filter规则：
```
filter {
  ...
  if [type] =~ "^node-*" {
    mutate {
      remove_field => [ "log_time" ]
    }
  }
}

```
重启服务后，果然再没看到`NumberFormatException`的异常再次抛出了，索引数量也恢复到正常水平。

![img](/img/in-post/post-NumberFormatException/post-NumFormEx-es-index-ok.jpeg)

*注：从图中还能看出其实今天这个问题早上8点新建索引的时候就出现了，直到10点半左右没顶住终于挂掉了。*

### 后记
对于某些历史遗留问题，或是很难推动的问题，与其僵死不动，不如另辟蹊径，想想有没有其他办法能够解决，毕竟办法总比困难多。