---
layout: post
title: "GlusterFS客户端无法挂载问题排查"
subtitle: "GlusterFS客户端挂载的两个小坑"
date: 2018-11-05 04:36PM
catalog: true
tags:
    - 运维
    - GlusterFS
---

### 背景

同事部署的GlusterFS集群，挂载的时候出现了些问题，这边记录下。

### 问题/缓解

#### 无法挂载

默认挂载命令敲下去过后，提示挂载失败：

```
# mount -t glusterfs <node1>:/replica-volume  /gluster/data
Mount failed. Please check the log file for more details.
```

查看日志，有不少调用函数失败的错误：

```
# cd /var/log/glusterfs/
# more glusterfs-data.log 
[2018-11-05 06:34:29.919194] I [MSGID: 100030] [glusterfsd.c:2441:main] 0-/usr/sbin/glusterfs: Started running /usr/sbin/glusterfs version 3.8.4 (args: /usr/sbin/glusterfs --volfile-server=<node1_ip> --
volfile-id=/replica-volume /glusterfs/data)
[2018-11-05 06:34:29.928423] I [MSGID: 101190] [event-epoll.c:602:event_dispatch_epoll_worker] 0-epoll: Started thread with index 1
[2018-11-05 06:34:29.932430] I [afr.c:94:fix_quorum_options] 0-replica-volume-replicate-0: reindeer: incoming qtype = none
[2018-11-05 06:34:29.932446] I [afr.c:116:fix_quorum_options] 0-replica-volume-replicate-0: reindeer: quorum_count = 2147483647
[2018-11-05 06:34:29.933469] I [MSGID: 101190] [event-epoll.c:602:event_dispatch_epoll_worker] 0-epoll: Started thread with index 2
[2018-11-05 06:34:29.933823] W [MSGID: 101174] [graph.c:360:_log_if_unknown_option] 0-replica-volume-readdir-ahead: option 'parallel-readdir' is not recognized
[2018-11-05 06:34:29.933853] W [MSGID: 101174] [graph.c:360:_log_if_unknown_option] 0-replica-volume-dht: option 'force-migration' is not recognized
[2018-11-05 06:34:29.933867] W [MSGID: 101174] [graph.c:360:_log_if_unknown_option] 0-replica-volume-replicate-0: option 'afr-pending-xattr' is not recognized
[2018-11-05 06:34:29.933939] W [MSGID: 101174] [graph.c:360:_log_if_unknown_option] 0-replica-volume-client-2: option 'transport.socket.keepalive-count' is not recognized
[2018-11-05 06:34:29.933998] W [MSGID: 101174] [graph.c:360:_log_if_unknown_option] 0-replica-volume-client-1: option 'transport.socket.keepalive-count' is not recognized
[2018-11-05 06:34:29.934056] W [MSGID: 101174] [graph.c:360:_log_if_unknown_option] 0-replica-volume-client-0: option 'transport.socket.keepalive-count' is not recognized
[2018-11-05 06:34:29.934080] I [MSGID: 114020] [client.c:2356:notify] 0-replica-volume-client-0: parent translators are ready, attempting connect on transport
[2018-11-05 06:34:29.936915] I [MSGID: 114020] [client.c:2356:notify] 0-replica-volume-client-1: parent translators are ready, attempting connect on transport
[2018-11-05 06:34:29.937308] I [rpc-clnt.c:2001:rpc_clnt_reconfig] 0-replica-volume-client-0: changing port to 49152 (from 0)
```

尝试在其他服务器上，但却没这个问题，最后查看了下glusterfs客户端版本，发现有问题服务器的版本较高（glusterfs-3.8.4），可以正常使用的版本是：glusterfs-3.7.1，这时候想到有问题的服务器使用的外部repo源，所以版本会比内部yum源更高，切换至内部源后问题解决。

```
# yum remove glusterfs glusterfs-fuse attr
# rpm -e --nodeps glusterfs-libs-3.8.4-54.15.el7.centos.x86_64
# rpm -e --nodeps glusterfs-client-xlators-3.8.4-54.15.el7.centos.x86_64
# yum downgrade libattr
# # UPDATE REPO
# yum install glusterfs glusterfs-fuse
...
Installed:
  glusterfs.x86_64 0:3.7.1-16.0.1.el7.centos                                                         glusterfs-fuse.x86_64 0:3.7.1-16.0.1.el7.centos                                                        

Dependency Installed:
  attr.x86_64 0:2.4.46-12.el7       glusterfs-client-xlators.x86_64 0:3.7.1-16.0.1.el7.centos       glusterfs-libs.x86_64 0:3.7.1-16.0.1.el7.centos       rsyslog-mmjsonparse.x86_64 0:7.4.7-12.el7      

Complete!
# mount -t glusterfs <node1>:/replica-volume /home/data/gfs
```

#### 挂载后无法创建文件

无论是创建文件，还是文件夹，均提示只读文件系统：

```
# cd /home/data/gfs/
# touch a.txt
touch: cannot touch ‘a.txt’: Read-only file system
# mkdir tmp
mkdir: cannot create directory ‘tmp’: Read-only file system
```

但服务端并无权限相关设置，再查日志：

```
# tail /var/log/glusterfs/home-data-gfs.log 
[2018-11-05 08:15:53.655725] E [name.c:242:af_inet_client_get_remote_sockaddr] 0-replica-volume-client-1: DNS resolution failed on host <node2>
[2018-11-05 08:15:53.656418] E [name.c:242:af_inet_client_get_remote_sockaddr] 0-replica-volume-client-2: DNS resolution failed on host <node3>
```

提示解析另外两台glusterfs域名失败，添加hosts后缓解。