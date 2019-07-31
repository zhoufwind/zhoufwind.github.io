---
layout: post
title: "k8s configmap配置问题导致容器崩溃"
date: 2019-07-31 11:49AM
catalog: true
tags:
    - 运维
    - K8S
---

### 背景

线上K8S ELK集群中的logstash及gohangout所在容器的配置文件均以`Config Maps`方式进行配置，昨晚K8S集群突然抛大量异常，挂了一批logstas/gohangout服务，进而导致es索引出现延迟。

![img](/img/in-post/post-190731-k8s-configmap/WechatIMG164.png)

### 问题

有些挂，有些没挂，是何原因？先以恢复业务为主，重启那些崩掉的服务，果然重启之后容器马上就拉起来了：

![img](/img/in-post/post-190731-k8s-configmap/WechatIMG168.png)

启动后单独查看pod信息，还是能够发现`FailedMount`，以及`MountVolume.SetUp failed for volume "config-log4j-k8s" : configmap references non-existent config key: gohangout-config-log4j-k8s`这样的异常信息，和前面截图上的报错一致。

```
# kubectl -n ns-elastic describe pod gohangout-log4j-k8s-66b49947d8-99xq9
...
    Mounts:
      /usr/share/gohangout from config-log4j-k8s (rw)
...
Volumes:
  config-log4j-k8s:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      gohangout-config
    Optional:  false
...
Events:
  Type     Reason       Age                  From                  Message
  ----     ------       ----                 ----                  -------
  Normal   Scheduled    24m                  default-scheduler     Successfully assigned ns-elastic/gohangout-log4j-k8s-66b49947d8-99xq9 to 10.10.52.75
  Normal   Pulling      24m                  kubelet, 10.10.52.75  Pulling image "dockerhub-wx.yeshj.com/ops/gohangout:1.2.7-8"
  Normal   Pulled       24m                  kubelet, 10.10.52.75  Successfully pulled image "dockerhub-wx.yeshj.com/ops/gohangout:1.2.7-8"
  Normal   Created      24m                  kubelet, 10.10.52.75  Created container gohangout-log4j-k8s
  Normal   Started      24m                  kubelet, 10.10.52.75  Started container gohangout-log4j-k8s
  Warning  FailedMount  105s (x19 over 24m)  kubelet, 10.10.52.75  MountVolume.SetUp failed for volume "config-log4j-k8s" : configmap references non-existent config key: gohangout-config-log4j-k8s
```

查了下`configmap`，这才发现了问题：

```
# kubectl -n ns-elastic get configmap
NAME               DATA   AGE
gohangout-config   2      77m
logstash-config    1      77m

# kubectl -n ns-elastic describe configmap gohangout-config
Name:         gohangout-config
Namespace:    ns-elastic
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"v1","data":{"gohangout-config-openresty-k8s":"inputs:\n    #- Stdin:\n    #    codec: json\n    - Kafka:\n        topic:\n ...

Data
====
gohangout-config-openresty-k8s:
----
...
gohangout-grok-patterns:
----
...

# kubectl -n ns-elastic describe configmap logstash-config
Name:         logstash-config
Namespace:    ns-elastic
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"v1","data":{"logstash-config-soagw-k8s":"input {\n  kafka {\n    codec =\u003e \"json\"\n    bootstrap_servers =\u003e \"ka...

Data
====
logstash-config-soagw-k8s:
----
...
```

我们发现configmap里只存有2个配置，而存着配置的pod没挂；挂的pod的configmap都没有显示出来，其用到的挂载卷：`config-log4j-k8s`并不存在于当前configmap中。

到这边大致就直到问题所在了，我们`logstash`/`gohangout`的configmap配置分别叫：`logstash-config`/`gohangout-config`，当时并没有单独为各个实例起不同的configmap name，所以导致了只有其中一个生效，其余的均被覆盖掉了。

### 缓解

要解决这个问题倒也简单，修改config name，增加一个后缀就ok了。

```
# kubectl -n ns-elastic get configmap
NAME                                         DATA   AGE
gohangout-config                             2      107m
gohangout-config-log4j                       1      8m27s
logstash-config                              1      107m
logstash-config-dns                          1      3m18s
logstash-config-gentian                      1      2m28s
logstash-config-iis                          1      107s
logstash-config-log4net-notifycenter-event   1      60s
logstash-config-named                        1      14s

# kubectl -n ns-elastic describe pod gohangout-log4j-k8s-69ccbdfb8c-qxrbf
...
    Mounts:
      /usr/share/gohangout from config-log4j-k8s (rw)
...
Volumes:
  config-log4j-k8s:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      gohangout-config-log4j
    Optional:  false
...
Events:          <none>
```

### 后记

但这边还引出了一个问题，如果存在覆盖的现象，为什么在启动pod时，为何并没有提示任何冲突信息，并且容器也能够正常运行？而是在正常运行了将近1个月的时间，突然崩溃？

我的理解是，启动一个pod服务时候，内存独享的，分配configmap的内存空间都是在单独的pod下。但暂时还没有找到明确崩溃的线索，只能猜测是k8s内部做了gc，把原来内存里存放configmap的那部分内容清理掉了，然后就崩了。

但我们想，如果一开始k8s就检测到configmap存在冲突，并予以提示，指出这样的配置是有问题的，需要调整configmap，以避免冲突，这样对用户来说也会更友好一些。