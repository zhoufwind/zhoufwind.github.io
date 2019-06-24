---
layout: post
title: "k8s集群升级时es集群运行状况记录"
date: 2019-06-24 04:04PM
catalog: true
tags:
    - 运维
    - ELK
    - K8S
---

### 背景

我司最大的一个es集群部署在k8s集群上，由于新版k8s能够解决不少业务上诸如响应超时等无解问题，于是升级k8s集群版本也算是一个常规操作了。

### 升级过程

es集群为有状态服务，重启k8s集群后，内部service会中断几秒钟，导致内部发现失败，es进程异常进而导致es集群崩溃。

在升级完毕后，由于通信恢复正常，es master及datanode分别进行重启，其中`master`通过`Deployment`方式进行部署，启动无序；`datanode`通过`StatefulSet`方式进行部署，启动顺序依次按照`elasticsearch-data-0`->`elasticsearch-data-1`->...->`elasticsearch-data-5`这种方式启动。

es集群节点（master及datanode）全部上线后，es读取索引目录下索引数据，由于我们为es集群部署了pvc挂载卷，所以老数据可以得以保留，新数据索引在老数据之上，es集群状态恢复正常。

以上为理想状况，但实际操作过程中，还是出现了2个问题。

### 问题

1. QA环境下，升级k8s集群后，es集群异常，所有es对外服务端口无法正常访问。

经查日志后发现以下异常日志：

```
journalctl -u kubelet
Jun 24 11:11:27 <hostname> kubelet[1450]: F0624 11:11:27.601871    1450 server.go:156] unknown flag: --allow-privileged
```

网上搜索后也确实在官网上搜到了相关的pr（#[77820][1]），其中提到的解决方式：

> ACTION REQUIRED: Deprecated Kubelet security controls AllowPrivileged, HostNetworkSources, HostPIDSources, HostIPCSources have been removed. Enforcement of these restrictions should be done through admission control instead (e.g. PodSecurityPolicy).  
> ACTION REQUIRED: The deprecated Kubelet flag `--allow-privileged` has been removed. Remove any use of `--allow-privileged` from your kubelet scripts or manifests.

调整k8s启动脚本后，问题得到缓解：

```
sed -i '/--allow-privileged=true/d' /etc/systemd/system/kubelet.service
```

2. 生产环境下，升级k8s集群后，当日某个索引数据丢失

正常状况下，es索引分片不应该丢失任何分片，但线上k8s集群升级后，在es集群恢复上线后，丢失了1个索引，这个索引在es集群启动后，被重新创建了，我们定义的template配置也没了，所以原本6分片0副本的配置恢复成了系统默认的5分片1副本。

猜想是由于重启k8s集群后，新启动的es集群读取历史索引数据速度慢于新索引数据创建的速度，也就是说，es已经准备要写入数据了，但当前索引数据中无该索引，并且`unassigned shards`中也没有或者还来不及load，导致这个索引直接被新建了。其他未丢失的索引数据是因为es已及时load了历史索引数据，亦或新索引数据进来的比较慢的原因。

针对这个问题，其实还是有解的，在升级k8s集群前，可以通过配置：`index.unassigned.node_left.delayed_timeout`来规避。这个配置告诉es集群，在分片数据脱离集群节点后若干时间内，不创建索引，也即分片数据脱离集群节点的超时时间。

另外，template配置应该是在`.kibana`这个索引中，怀疑也是同样的原因，导致新索引数据覆盖了老索引数据，造成老template配置丢失。

[1]: https://github.com/kubernetes/kubernetes/pull/77820 "Remove deprecated Kubelet security controls"
