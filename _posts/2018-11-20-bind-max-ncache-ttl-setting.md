---
layout: post
title: "某机房缓存dns递归查询无响应问题排查"
subtitle: "BIND域名不存在缓存时间配置"
date: 2018-11-20 03:16PM
catalog: true
tags:
    - 运维
    - DNS
    - BIND
---

### 背景

有同学反馈在添加域名解析后，某机房（这里代称：机房A）解析无返回，接到反馈后立刻开始排查。

### 故障定位

首先查看dns管理平台，域名解析添加完毕，并且单独针对机房A单独配置了区域解析。

分别在办公室本机、机房A以及机房B（另外一个机房，代称：机房B）进行dig操作，本机及机房B解析结果均正常返回，唯独机房A无响应，结果说明：

- 权威dns运行正常，至少机房B的缓存dns及办公室本机的localdns均返回解析结果了
- 机房A解析无响应，应该是机房A缓存dns的独立问题

单独指定机房A的两台缓存dns做`dig`查询，发现均可复现该问题。

秉承重启大法，先重启了其中一台缓存dns，再次指定这台缓存dns做解析操作，结果立即返回，由此推测应该是bind服务的问题。

### 原因分析

查看bind日志（`/var/log/messages`），并未发现明显异常错误，说明bind本身应该是正常的，否则解析其他域名也应该有问题。

比较了下机房A缓存dns及机房B缓存dns的配置文件（`/etc/named.conf`），发现了可疑点：

机房B的缓存dns单独配有一个参数：`max-ncache-ttl 60;`，而机房A的缓存dns没有，这个参数的作用在谷歌上查询了下，[解释][1]如下：

> max-ncache-ttl sets the maximum time (in seconds) for which the server will cache negative (NXDOMAIN) answers (positives are defined by max-cache-ttl ). The default max-ncache-ttl is 10800 seconds (3 hours).

由此，基本可以推断问题原因：

用户添加了一条域名解析记录至权威dns master，由于主从同步存在延迟，权威dns master尚未将记录同步至所有权威dns slave服务器，这个时候，机房A的缓存dns向权威dns（slave）发起递归请求，权威dns（slave）查找后无记录，所以返回域名不存在，而这个状态被机房A的缓存dns缓存下来了，并且默认缓存3小时，也就是说，在3小时之内再向机房A缓存dns请求该域名，始终都会是`NXDOMAIN`的状态，最终导致用户解析无返回。

### 故障复现

修改了机房A其中一台缓存dns，增加了`max-ncache-ttl 60;`配置，重启使之生效。

- 新建了一个测试域名，在刚增加完域名解析记录的一瞬间，对修改了配置的缓存dns发起请求，返回无结果，隔了1分钟再去解析，可以正常返回。

- 再新建一个测试域名，在刚添加完域名解析记录的一瞬间，对未修改配置的缓存dns发起请求，返回无结果，隔了1小时再去解析，仍然无结果返回。

由此证明推断成立。

### 缓解

修改机房A两台缓存dns的`/etc/named.conf`配置文件，添加`max-ncache-ttl 60;`参数，重启生效。

### 小结

小洞不补，大洞吃苦！线上各机房系统及应用服务配置必须进行统一，尤其是类似dns这种基础组建服务，否则很容易出现各种妖异问题，耗费排障时间。

[1]: https://lists.isc.org/pipermail/bind-users/2016-March/096409.html "what does "max-ncache-ttl 0;" mean?"