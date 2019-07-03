---
layout: post
title: "corosync+pacemaker脑裂问题处理"
date: 2019-07-03 05:06PM
catalog: true
tags:
    - 运维
    - Linux
    - 高可用
---

### 背景

办公网DNS使用Nginx作为反向代理服务器，服务器配有corosync+pacemaker，用来实现高可用。

### 问题

今天在维护这几台服务器时，突然发现2台Nginx的corosync发生了脑裂的现象，各自都分配了vip：

- NodeA

```
# crm status
Stack: classic openais (with plugin)
Current DC: pr-v17-122.hjidc.com (version 1.1.15-5.el6-e174ec8) - partition WITHOUT quorum
Last updated: Wed Jul  3 16:06:32 2019		Last change: Sat Mar 30 02:36:27 2019 by root via cibadmin on pr-v17-128.hjidc.com
, 2 expected votes
2 nodes and 2 resources configured

Online: [ pr-v17-122.hjidc.com ]
OFFLINE: [ pr-v17-128.hjidc.com ]

Full list of resources:

 vip_192.168.18.244	(ocf::heartbeat:IPaddr):	Started pr-v17-122.hjidc.com
 vip_192.168.18.245	(ocf::heartbeat:IPaddr):	Started pr-v17-122.hjidc.com

# ip a
...
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:55:00:00:67:e9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.18.177/24 brd 192.168.18.255 scope global eth1
    inet 192.168.18.244/24 brd 192.168.18.255 scope global secondary eth1:vip83
    inet 192.168.18.245/24 brd 192.168.18.255 scope global secondary eth1:vip84
    inet6 fe80::5055:ff:fe00:67e9/64 scope link 
       valid_lft forever preferred_lft forever

```

- NodeB

```
# crm status
Stack: classic openais (with plugin)
Current DC: pr-v17-128.hjidc.com (version 1.1.15-5.el6-e174ec8) - partition WITHOUT quorum
Last updated: Wed Jul  3 16:07:03 2019		Last change: Sat Mar 30 02:36:18 2019 by root via cibadmin on pr-v17-128.hjidc.com
, 2 expected votes
2 nodes and 2 resources configured

Online: [ pr-v17-128.hjidc.com ]
OFFLINE: [ pr-v17-122.hjidc.com ]

Full list of resources:

 vip_192.168.18.244	(ocf::heartbeat:IPaddr):	Started pr-v17-128.hjidc.com
 vip_192.168.18.245	(ocf::heartbeat:IPaddr):	Started pr-v17-128.hjidc.com

# ip a
...
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:55:00:00:69:c8 brd ff:ff:ff:ff:ff:ff
    inet 192.168.18.179/24 brd 192.168.18.255 scope global eth1
    inet 192.168.18.244/24 brd 192.168.18.255 scope global secondary eth1:vip83
    inet 192.168.18.245/24 brd 192.168.18.255 scope global secondary eth1:vip84
    inet6 fe80::5055:ff:fe00:69c8/64 scope link 
       valid_lft forever preferred_lft forever
```

分别重启两台服务器都corosync和pacemaker服务，并没有恢复，又查了日志，没查到明显报错。

### 缓解

查看crm配置，发现多处配置通过域名方式调用服务器，但ping域名后解析出来的ip地址与实际服务器ip地址不符，修改`hosts`文件，指定域名到服务器ip地址，并重启corosync+pacemaker后问题得到缓解。

- NodeA

```
# crm status
Stack: classic openais (with plugin)
Current DC: pr-v17-122.hjidc.com (version 1.1.15-5.el6-e174ec8) - partition with quorum
Last updated: Wed Jul  3 17:16:51 2019		Last change: Wed Jul  3 16:18:37 2019 by root via cibadmin on pr-v17-128.hjidc.com
, 2 expected votes
2 nodes and 2 resources configured

Online: [ pr-v17-122.hjidc.com pr-v17-128.hjidc.com ]

Full list of resources:

 vip_192.168.18.244	(ocf::heartbeat:IPaddr):	Started pr-v17-122.hjidc.com
 vip_192.168.18.245	(ocf::heartbeat:IPaddr):	Started pr-v17-128.hjidc.com

# ip a
...
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:55:00:00:67:e9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.18.177/24 brd 192.168.18.255 scope global eth1
    inet 192.168.18.244/24 brd 192.168.18.255 scope global secondary eth1:vip83
    inet6 fe80::5055:ff:fe00:67e9/64 scope link 
       valid_lft forever preferred_lft forever

```

- NodeB

```
# crm status
Stack: classic openais (with plugin)
Current DC: pr-v17-122.hjidc.com (version 1.1.15-5.el6-e174ec8) - partition with quorum
Last updated: Wed Jul  3 17:17:00 2019		Last change: Wed Jul  3 16:18:37 2019 by root via cibadmin on pr-v17-128.hjidc.com
, 2 expected votes
2 nodes and 2 resources configured

Online: [ pr-v17-122.hjidc.com pr-v17-128.hjidc.com ]

Full list of resources:

 vip_192.168.18.244	(ocf::heartbeat:IPaddr):	Started pr-v17-122.hjidc.com
 vip_192.168.18.245	(ocf::heartbeat:IPaddr):	Started pr-v17-128.hjidc.com

# ip a
...
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:55:00:00:69:c8 brd ff:ff:ff:ff:ff:ff
    inet 192.168.18.179/24 brd 192.168.18.255 scope global eth1
    inet 192.168.18.245/24 brd 192.168.18.255 scope global secondary eth1:vip84
    inet6 fe80::5055:ff:fe00:69c8/64 scope link 
       valid_lft forever preferred_lft forever
```

### 原因

原先机房linux服务器都使用了`.hjidc.com`作为服务器后缀，并且解析都没有问题，而现在，所有主机名解析都会向`hjidc.com`这个权威域进行解析，解析出来的结果自然与实际服务器ip地址不一致。猜想应该是`hjidc.com`这个权威域的管理员配置了默认解析，才导致了脑裂的问题。