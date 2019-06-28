---
layout: post
title: "k8s-elastic（利用k8s部署elk集群）版本升级记录（v6.5.1 -> v7.2.0）"
date: 2019-06-28 05:28PM
catalog: true
tags:
    - 运维
    - ELK
    - K8S
---

### 背景

随着elk版本不断升级，特别是4月份es出了7.0大版本，到最近的7.2.0小版本，有了很多实用新功能，这边准备进行升级。

由于我司全面推广k8s项目，所有新上项目都会考虑部署至k8s集群，因此本次升级也是在k8s环境下进行的。

### 升级步骤

- 官网上分别拉取elasticsearch/kibana/logstash最新7.2.0版本镜像，修改tag，并上传至公司私有镜像。
- 修改yml文件，删除原先容器，开启新容器，完成升级。

### 遇到的坑

当然，升级过程没这么简单，遇到了几个问题点在这里列下。

#### 集群探测配置

新版废弃了原先`discovery.zen.ping.unicast.hosts`及`discovery.zen.minimum_master_nodes`的探测方式，而是改为了`discovery.seed_hosts`及`cluster.initial_master_nodes`，其中第一项换汤不换药，值还是一样的，我们还是填内部服务域名，而后一项就有点恶心了，这项在实体服务器上配置没任何问题，把符合master节点的ip或主机名配进去就可以了，但在k8s环境下，master节点的ip是pod ip，随即分配的，主机名由于是`deployment`下部署的，主机名的尾巴也是随机字符，同样无法指定，不配的话各节点服务器无法选举master，无法组成集群：

> {"type": "server", "timestamp": "2019-06-26T11:03:02,636+0000", "level": "WARN", "component": "o.e.c.c.ClusterFormationFailureHelper", "cluster.name": "elasticsearch-cluster-v7", "node.name": "elasticsearch-master-8b5bc975b-vdb94",  "message": "**master not discovered yet, this node has not previously joined a bootstrapped (v7+) cluster**, and this node must discover master-eligible nodes [HOSTNAME] to bootstrap a cluster: have discovered []; discovery will continue using [192.153.20.100:9300] from hosts providers and [{elasticsearch-master-8b5bc975b-vdb94}{HMo1b8jfSXSNtl1VSTlMsg}{yCQqrg3xTcqO7eXQr3sEnQ}{192.152.13.233}{192.152.13.233:9300}{ml.machine_memory=33728757760, xpack.installed=true, ml.max_open_jobs=20}] from last-known cluster state; node term 0, last-accepted version 0 in term 0"  }

![img](/img/in-post/post-190628-k8s-elastic-upgrade/WechatIMG3009.jpeg)

针对这个问题，我们将es-master部署方式由`depolyment`调整至`statefulset`，这样就解决了主机名无法指定的问题，配置如下：

```
            - name: cluster.initial_master_nodes
              value: "elasticsearch-master-0,elasticsearch-master-1,elasticsearch-master-2"
```

master成功探测对方的日志如下：

> {"type": "server", "timestamp": "2019-06-27T10:30:40,996+0000", "level": "WARN", "component": "o.e.c.c.ClusterFormationFailureHelper", "cluster.name": "elasticsearch-cluster-v7", "node.name": "**elasticsearch-master-2**",  "message": "master not discovered yet, this node has not previously joined a bootstrapped (v7+) cluster, and this node must discover master-eligible nodes [elasticsearch-master-service] to bootstrap a cluster: **have discovered [{elasticsearch-master-1}**{EnGhzi7JRLmDkaYuZKuADg}{t0kR5vjaSsKpOfbptBbZqQ}{192.152.203.168}{192.152.203.168:9300}{ml.machine_memory=33728757760, ml.max_open_jobs=20, xpack.installed=true}, {elasticsearch-master-0}{CZ60XOP3QwSJth9Fp2gWzA}{7QM1yDB_SKCHv5wEPlaXBw}{192.152.13.201}{192.152.13.201:9300}{ml.machine_memory=33728757760, ml.max_open_jobs=20, xpack.installed=true}]; discovery will continue using [192.153.144.253:9300] from hosts providers and [{elasticsearch-master-2}{GxnZZFVUSB2rJwAkA9sGfw}{rX8ha6MfQaq4GjF80MOTeA}{192.152.53.241}{192.152.53.241:9300}{ml.machine_memory=33728757760, xpack.installed=true, ml.max_open_jobs=20}] from last-known cluster state; node term 0, last-accepted version 0 in term 0"  }

#### master及data集群uuid冲突问题

上个问题解决后，又遇到了新的问题，master集群节点可正常启动，但是data节点启动后，无法加入master集群，查看日志显示`uuid`不匹配：

> "Caused by: org.elasticsearch.cluster.coordination.CoordinationStateRejectedException: **join validation on cluster state with a different cluster uuid bb79xFABQAq_gkKEc1TTpw than local cluster uuid DekASqJ2TIOtB12Q6KeZEw**, rejecting"

![img](/img/in-post/post-190628-k8s-elastic-upgrade/WechatIMG3011.jpeg)

后经测试，取消了es-data节点的pvc配置，使用容器内部存储，这个问题就解决了。但是我们必须用到持久化存储，pvc不能丢弃，于是我们便尝试将es-master集群也使用pvc的方式部署（原先部署es-master图省事，使用容器内部存储），看能不能达到同样效果，最后证实可行！

解决了这两个问题后，新版es集群也即部署好了，kibana及logstash则较为简单，配置不用便，修改下镜像版本就可以了。

### 参考文档

- [通过环境变量将Pod信息呈现给容器](https://kubernetes.io/zh/docs/tasks/inject-data-application/environment-variable-expose-pod-information/)  
- [Master not discovered yet, this node has not previously joined a bootstrapped (v7+) cluster](https://discuss.elastic.co/t/master-not-discovered-yet-this-node-has-not-previously-joined-a-bootstrapped-v7-cluster/176304)  
- [elasticsearch-kubed](https://github.com/jswidler/elasticsearch-kubed)