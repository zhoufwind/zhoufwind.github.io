---
layout: post
title: "Elastic数据类型转换记录"
subtitle: "Elastic Fields type转换（string到number）"
date: 2018-09-28 11:09AM
catalog: true
tags:
    - 运维
    - ELK
---

### 背景

ELK接Nginx日志需求，日志已成功接入，但Kibana上显示`sc_bytes`字段为`string`，无法进行聚合操作，无法为该字段配置曲线图。

### 问题

查看logstash的`grok`规则，单独为其配置转换规则后，重启服务后仍然无效：

```
  mutate {
    convert => ["sc_bytes", "float"]
  }
```

查看索引的数据结构，发现非`string`，也非`float`，而是什么`keyword`字段类型：

```
# curl -s http://<es_ip>:9200/logstash-kafka-evops-2018.09.28
...
    "sc_bytes": {
        "type": "text",
        "norms": false,
        "fields": {
            "raw": {
                "type": "keyword",
                "ignore_above": 256
            }
        }
    },
...
```

然而，查看了[ES官网文档][1]后，2.x之后版本不支持`keyword`类型，会自动降级至`string`类型，从而造成了我们再Kibana上看到`sc_bytes`字段为`string`：

> Indexes imported from 2.x do not support keyword. Instead they will attempt to downgrade `keyword` into `string`. 

### 缓解

让ES自动识别成`number`类型失败，我们通过初始化索引模板的方式来解决这个问题。

我们通过[`cerebro`][2]这个小工具来进行配置：

```
# Section
index templates

# Template name
template-evops-nginx

# Template content
{
  "order": 1,
  "template": "logstash-kafka-evops-*",
  "settings": {
    "index": {
      "number_of_shards": "2",
      "number_of_replicas": "1"
    }
  },
  "mappings": {
    "_default_": {
      "properties": {
        "sc_bytes": {
          "type": "double"
        }
      }
    }
  },
  "aliases": {}
}
```

删除老索引，新索引自动创建后，字段类型转换完成：

```
# curl -XDELETE http://<es_ip>:9200/logstash-kafka-evops-2018.09.28

# curl -s http://<es_ip>:9200/logstash-kafka-evops-2018.09.28
...
    "sc_bytes": {
        "type": "double"
    },
...
```

### 后记

这个已经是老问题了，这两天接到新接入需求，又遇到这个问题，顺便在这边记录下。

[1]: https://www.elastic.co/guide/en/elasticsearch/reference/current/keyword.html "Keyword datatypeedit"
[2]: https://github.com/lmenezes/cerebro