---
layout: post
title: "iptables forward drop策略导致k8s网络异常"
subtitle: "k8s网络转发失败故障排查记录"
date: 2019-01-28 11:22AM
catalog: true
tags:
    - 运维
    - K8S
    - Linux
---

### 问题

新部署的k8s集群存在一个问题，部署完服务后，外部访问有时能访问，有时不能访问。经手工排查后，发现在访问容器内服务时，只能访问创建在本机pod内容器的服务，跨node节点访问srv都会失败。

看表象似乎是calico网络通信的问题，但奇怪的是跨node节点间ping cluster ip都是通的，又尝试网络抓包定位问题，node节点数据报文发出后，无数据包返回。

在排除了操作系统（`net.ipv4.ip_forward = 1`）、calico、kube-proxy，甚至是网络权限之后，怀疑是iptables数据转发层面上的问题。

### 缓解

以下为故障时的iptables，可发现`*filter`中`:FORWARD DROP [0:0]`这条规则是有问题的，此条规则为k8s程序自己配置的，但forward被drop后会导致网络数据无法进行转发。

```
# iptables-save
...
# Generated by iptables-save v1.4.21 on Sat Jan 26 04:26:04 2019
*filter
:INPUT ACCEPT [6826:1102864]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [6889:1824275]
COMMIT
```

在输入`iptables -P FORWARD ACCEPT`后，问题缓解。

更新后的iptables规则如下：

```
# iptables-save
...
# Generated by iptables-save v1.4.21 on Mon Jan 28 21:50:39 2019
*filter
:INPUT ACCEPT [3119:638080]
:FORWARD ACCEPT [26:1432]
:OUTPUT ACCEPT [3236:612347]
COMMIT
```

### 相关补充

#### iptables 
排查iptables比较常用的[两个命令][1]：
1. `iptables-save`
2. `iptables -t nat -nvL`

iptables[常用规则][2]：
格式：iptables [-t table] -P [INPUT,OUTPUT,FORWARD] [ACCEPT,DROP ]
-p : 定义策略(Policy)。注意:P为大写 
ACCEPT:数据包可接受 
DROP：数据包被丢弃，client不知道为何被丢弃。

Calico网络的原理、组网方式与使用[链接][3]。

### 后记

默认`FORWARD`默认设置为`ACCEPT`，所以我们推测是k8s自己调整了iptables中filter的forward策略。

这个问题排查了比较长时间，主要原因还是对iptables的理解不深，虽然大体知道是网络数据转发方面的问题，但并未深入排查系统转发策略，此次问题加深了对系统网络层转发方面问题排查的经验。

[1]: https://www.cnblogs.com/charlieroro/p/9588019.html "理解kubernetes环境的iptables"
[2]: https://blog.csdn.net/canot/article/details/51289176 "Centos6.5 iptables的Filter详解"
[3]: https://www.lijiaocn.com/%E9%A1%B9%E7%9B%AE/2017/04/11/calico-usage.html "Calico网络的原理、组网方式与使用"