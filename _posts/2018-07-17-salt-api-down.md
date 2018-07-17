---
layout: post
title: "由salt-api并发瓶颈所导致的线上故障分析"
subtitle: "锅是你的，跑也跑不掉！"
date: 2018-07-18 12:06 +0800
catalog: true
tags:
    - 运维
    - Salt
    - TCP/IP
---
### 背景
距离上次更新已经过去了大半周，这大半周时间主要用来填上周遇到的一个坑，没错，就是上篇博文最后提到的那个线上故障。

当天晚上并未查明具体问题原因，第二天白天，出现异常问题程序（暂称：程序A）的开发同学终于醒了……（当天晚上睡着了，电话死活没打通），后排查发现是调用salt-api异常，微信群里反馈连接salt有问题。

看到这边，突然感到凉凉，昨晚排查了一晚上的salt-master，观察其状态正常，就认为salt没问题，直接反应是程序A调用方的问题，完全忽略了salt-api这一环，瞬间崩溃。

不过我这边也得喊个冤，salt-api这块的运维是上一辈已经离职的同学留下来的产物，我这边除了上次支援安全组的需要，改过一次salt-api的密码后，其他的都没接手过，真的能用一句“锅从天上来”来形容……

### 排查过程

#### 重启salt-api服务
不管salt-api啥问题，本着先解决线上问题的原则，直接敲命令重启：
```
# service salt-api restart
```

完了查看salt-api日志，发现salt-api日志抛错：
```
2018-07-13 09:27:51,151 [salt.utils.process                       ][INFO    ][12603] Process <function start at 0x328e9b0> (14351) died with exit status None, restarting...
2018-07-13 09:27:52,155 [salt.utils.process                       ][DEBUG   ][12603] Started 'salt.loaded.int.netapi.rest_cherrypy.start' with pid 14374
2018-07-13 09:27:52,162 [salt.utils.event                         ][DEBUG   ][14374] MasterEvent PUB socket URI: ipc:///var/run/salt/master/master_event_pub.ipc
2018-07-13 09:27:52,162 [salt.utils.event                         ][DEBUG   ][14374] MasterEvent PULL socket URI: ipc:///var/run/salt/master/master_event_pull.ipc
2018-07-13 09:27:52,164 [salt.utils.event                         ][DEBUG   ][14374] Sending event - data = {'_stamp': '2018-07-13T01:27:52.163387'}
2018-07-13 09:27:52,185 [cherrypy.error                           ][INFO    ][14374] [13/Jul/2018:09:27:52] ENGINE Listening for SIGHUP.
2018-07-13 09:27:52,185 [cherrypy.error                           ][INFO    ][14374] [13/Jul/2018:09:27:52] ENGINE Listening for SIGTERM.
2018-07-13 09:27:52,185 [cherrypy.error                           ][INFO    ][14374] [13/Jul/2018:09:27:52] ENGINE Listening for SIGUSR1.
2018-07-13 09:27:52,185 [cherrypy.error                           ][INFO    ][14374] [13/Jul/2018:09:27:52] ENGINE Bus STARTING
2018-07-13 09:27:52,186 [cherrypy.error                           ][INFO    ][14374] [13/Jul/2018:09:27:52] ENGINE Started monitor thread '_TimeoutMonitor'.
2018-07-13 09:27:57,207 [cherrypy.error                           ][ERROR   ][14374] [13/Jul/2018:09:27:57] ENGINE Error in 'start' listener <bound method Server.start of <cherrypy._cpserver.Server object at 0x3400d90>>
Traceback (most recent call last):
  File "/usr/lib/python2.6/site-packages/cherrypy/process/wspbus.py", line 197, in publish
    output.append(listener(*args, **kwargs))
  File "/usr/lib/python2.6/site-packages/cherrypy/_cpserver.py", line 151, in start
    ServerAdapter.start(self)
  File "/usr/lib/python2.6/site-packages/cherrypy/process/servers.py", line 167, in start
    wait_for_free_port(*self.bind_addr)
  File "/usr/lib/python2.6/site-packages/cherrypy/process/servers.py", line 410, in wait_for_free_port
    raise IOError("Port %r not free on %r" % (port, host))
IOError: Port 443 not free on '0.0.0.0'

2018-07-13 09:27:57,208 [cherrypy.error                           ][ERROR   ][14374] [13/Jul/2018:09:27:57] ENGINE Shutting down due to error in start listener:
Traceback (most recent call last):
  File "/usr/lib/python2.6/site-packages/cherrypy/process/wspbus.py", line 235, in start
    self.publish('start')
  File "/usr/lib/python2.6/site-packages/cherrypy/process/wspbus.py", line 215, in publish
    raise exc
ChannelFailures: IOError("Port 443 not free on '0.0.0.0'",)

2018-07-13 09:27:57,208 [cherrypy.error                           ][INFO    ][14374] [13/Jul/2018:09:27:57] ENGINE Bus STOPPING
2018-07-13 09:27:57,208 [cherrypy.error                           ][INFO    ][14374] [13/Jul/2018:09:27:57] ENGINE HTTP Server cherrypy._cpwsgi_server.CPWSGIServer(('0.0.0.0', 443)) already shut down
2018-07-13 09:27:57,208 [cherrypy.error                           ][INFO    ][14374] [13/Jul/2018:09:27:57] ENGINE Stopped thread '_TimeoutMonitor'.
2018-07-13 09:27:57,208 [cherrypy.error                           ][INFO    ][14374] [13/Jul/2018:09:27:57] ENGINE Bus STOPPED
2018-07-13 09:27:57,209 [cherrypy.error                           ][INFO    ][14374] [13/Jul/2018:09:27:57] ENGINE Bus EXITING
2018-07-13 09:27:57,209 [cherrypy.error                           ][INFO    ][14374] [13/Jul/2018:09:27:57] ENGINE Bus EXITED
```

看到这个报错，感觉像是443端口分配失败的样子，`ps -ef|grep salt-api`看了下进程状态，糟，确实有问题，老的salt-api父进程没了，子进程还在，怪不得新的进程跑不起来了：
```
# ps -ef|grep salt-api|grep -v grep
root     12603     1  0 09:25 ?        00:00:00 /usr/bin/python2.6 /usr/bin/salt-api -d --log-file=/var/log/salt/api --log-level=debug ProcessManager
root     16825 12603  1 09:32 ?        00:00:00 /usr/bin/python2.6 /usr/bin/salt-api -d --log-file=/var/log/salt/api --log-level=debug ProcessManager
root     17502     1 99 Jul12 ?        2-04:45:42 /usr/bin/python2.6 /usr/bin/salt-api -d --log-file=/var/log/salt/api --log-level=debug
```

猜测进程应该是挂死了，只能先kill -9把新老进程一起杀掉，再尝试启动：
```
# kill -9 17502
# service salt-api stop
Stopping salt-api daemon:                                  [  OK  ]
# service salt-api status
salt-api is stopped
# ps -ef|grep salt-api|grep -v grep
root     17248     1 20 09:32 ?        00:00:06 /usr/bin/python2.6 /usr/bin/salt-api -d --log-file=/var/log/salt/api --log-level=debug ProcessManager
# kill -9 17248
# service salt-api start
```

这一波操作后，salt-api才恢复工作，问题暂时得到缓解。

然而，这才是问题的开始，一些列为什么随之而来。

#### Q

那么，问题来了：
- 为什么是salt-api的问题？
- 为什么故障时没发现是salt-api的问题？
- 为什么重启salt-api会失败？
- 为什么salt-api会挂死？
- 为什么没有监控到？

#### A

- 为什么是salt-api的问题？

程序A是通过salt-api执行下发指令操作的，salt-api相当于发送命令的通信员，通信员挂了，命令自然无法下达，造成程序不可用。

- 为什么故障时没发现是salt-api的问题？

对可能引起程序A异常的原因不了解，对salt-api的不熟悉所导致。

- 为什么重启salt-api会失败？

这个涉及具体salt-api底层执行逻辑，现象是：当并发达到瓶颈后，salt-api服务状态不稳定，先开始出现大量延迟响应，后直接拒绝服务。默认`/etc/init.d/salt-api restart`调用kill 15，不会真正中断程序进程，只会通知salt-api自行关闭，但salt-api此时已经挂死，无法接受任何响应，所以无法杀干净，造成重启失败。

- 为什么salt-api会挂死？

这个问题提到了重点，这边观察下来是由于上周有其他调用salt-api的程序B更新了版本，更新版本后增加了调用salt-api的频率，从而增加了salt-api的并发压力。从salt-api的服务器上观察，CPU、内存、网络流量等均无明显异常，唯独TCP连接数有较大异常：

salt-api服务器的tcp close wait数量急剧上升：
![img](/img/in-post/post-salt-api-down/salt-api-tcp-close-wait-high.jpeg)

程序B的tcp time wait数量翻倍：
![img](/img/in-post/post-salt-api-down/cmdb-tcp-time-wait-high.jpeg)

我们认为，他们间的相互关系是：
程序B调用salt-api的频率增加，导致salt-api的并发增加，并达到了salt-api的处理瓶颈，随后造成阻塞，响应出现延迟，程序B的调用任务并未减少，造成雪崩，最终导致salt-api的拒绝服务，进程挂死。

这边引用TCP/IP 4次握手关闭，可理解为：
1. 程序B执行的是同步任务，设有超时时间，超过超时时间后salt-api仍未响应，程序B主动关闭，发FIN；
2. salt-api服务端收到FIN，salt-api服务被动关闭，但此时salt-api由于进程挂死，造成阻塞，形成大量close wait；

![img](/img/in-post/post-salt-api-down/tcpclose_thumb.png)

- 为什么没有监控到？

之前配过salt-api基本认证成功的监控项，但当salt-api进程挂死状态下，该监控超时无数据，而监控报警的策略是非0报警，造成了没有报警产生。

### 改进
针对这个问题，后续采取了2个方式进行优化：
- 升级salt-api版本，避免由于salt性能问题导致的低并发，包括salt-api引用到的cherrypy；
- 调整程序B对salt-api的调用方式，由原来的同步执行改为异步执行，间接减少并发量。

调整过后，这边已经观察了1天，再未出现salt-api响应异常的问题，close wait数量保持在0。

升级salt-api必须同时升级salt框架包，也就意味着salt-master/ salt-syndic/ salt-minion需要一起跟着升级，升级过程不详细说了，过程相当紧张，担心升级后的可用性及兼容性问题，好在最后有惊无险，平安过渡，也算是松了一口气。更令人兴奋的是，原本执行一些消耗salt-master性能的任务，似乎现在也不再是问题了，salt-master稳定性一并得到了提升。

### 后记
本次故障首先发现问题的时间太慢，不能再问题出现时立即推导出问题根源；另外排查问题速度也较慢，今后一定需要多联想，思维不能太固定狭隘，多方面考虑问题能够更效率的发现并解决问题。