---
layout: post
title: "rsync文件同步失败问题排查记录"
subtitle: "弱网环境导致rsync文件同步失败"
date: 2018-10-26 01:57PM
catalog: true
tags:
    - 运维
    - Rsync
    - TCP/IP
---

### 背景

周三下午收到第一例用户反馈发布qa环境失败，周四又出三例，同样都是qa环境发布，感觉有坑，立刻排查。

### 问题

#### rsync客户端：抛错异常退出

发布失败截图如下：

![img](/img/in-post/post-181026-rsync-error-timeout/WechatIMG2284.jpeg)

看报错信息应该是rsync同步文件失败，并非salt问题，为确认这点，我们登录salt-master，手工执行rsync命令，确实无返回，排除salt及salt-api问题。

登录同步异常服务器，手工执行rsync同步命令，可以复现：

```
# rsync -avz --delete --exclude='.git' --exclude='.svn' rsync://<rsync_srv>:<rsync_port>/path/to/folder /tmp/rsync-test
receiving incremental file list
...
rsync: read error: Connection reset by peer (104)
rsync error: error in rsync protocol data stream (code 12) at io.c(759) [receiver=3.0.6]
rsync: connection unexpectedly closed (99 bytes received so far) [generator]
rsync error: error in rsync protocol data stream (code 12) at io.c(600) [generator=3.0.6]
```

#### rsync客户端：进程僵死

周四反馈发布失败的同学情况又不一样了，发布并未抛明显异常，但是进度条卡住一直无返回，查询salt-master日志后，rsync命令成功下发，但确实无结果返回，怀疑服务器执行rsync命令失败，进程僵死，手工上服务器执行了把，确实如此，只有当手工终止rsync命令后（ctrl+c），才抛异常信息：

```
# rsync -avzP --delete  --exclude='.git' --exclude='.svn' rsync://<rsync_srv>:<rsync_port>/path/to/folder /tmp/rsync-test
Password: 
receiving incremental file list
./
<rsync-test-pkg>-SNAPSHOT.jar
^C
rsync error: received SIGINT, SIGTERM, or SIGHUP (code 20) at rsync.c(551) [generator=3.0.9]

rsync error: received SIGUSR1 (code 19) at main.c(1298) [receiver=3.0.9]
```

执行第二，甚至第三次时，才成功：

```
# rsync -avzP --delete  --exclude='.git' --exclude='.svn' rsync://<rsync_srv>:<rsync_port>/path/to/folder /tmp/rsync-test
Password: 
receiving incremental file list
./
<rsync-test-pkg>-SNAPSHOT.jar
    60606801 100%   16.13MB/s    0:00:03 (xfer#1, to-check=0/3)

sent 21035 bytes  received 43167123 bytes  5758421.07 bytes/sec
total size is 60607326  speedup is 1.40
```

网上搜了下，发现已经有人发现rsync类似问题了，引用其[博客][1]:

> 尽管您可能已经在rsyncd服务的后端进程中设置了--timeout选项(即在rsyncd.conf配置中)，然而，在某些情况下（under the circumstances），这个选项可能根本不起作用，一些极不稳定的网络导致大量TCP超时连接，进而导致rsync进程失败，虽然断裂的TCP连线已经消失，但rsync应用进程却可能因为种种原因（如因等候I/O中断而处于不可中断状态），而遗留在系统之中，并最终变成为僵尸进程（zombie process）。

> 按照其手册页的解释，rsync命令本身的timeout预设为0，也就是没有逾时设置，因此运行中的rsync进程将会永久地等待远端的反应。在rsyncd服务后端进程的 rsyncd.conf中设置timeout选项，同时在rsync客户端命令行中使用timeout选项，实践证明是可杜绝此问题的。

于是在服务器上添加`--timeout`参数，再次执行后确实能够异常退出了：

```
# rsync -avzP --timeout=60 --delete  --exclude='.git' --exclude='.svn' rsync://<rsync_srv>:<rsync_port>/path/to/folder /tmp/rsync-test
Password: 
receiving incremental file list
./
<rsync-test-pkg>-SNAPSHOT.jar
    20114253  33%   19.18MB/s    0:00:02
[receiver] io timeout after 60 seconds -- exiting
rsync error: timeout in data send/receive (code 30) at io.c(140) [receiver=3.0.9]
rsync: connection unexpectedly closed (115 bytes received so far) [generator]
rsync error: error in rsync protocol data stream (code 12) at io.c(605) [generator=3.0.9]
```

#### rsync服务端：异常日志

怀疑过是不是rsync服务端问题，重启过服务端rsync服务，问题依然存在；换过rsync服务端，问题依然存在，由此可以排除服务端问题。

但无论客户端抛哪种问题，服务端均能捕捉到，并输出相应日志：

```
2018/10/26 14:40:30 [4228] name lookup failed for <rsync-client>: Name or service not known
2018/10/26 14:40:30 [4228] connect from UNKNOWN (<rsync-client>)
2018/10/26 14:40:30 [4228] rsync on path/to/folder from UNKNOWN (<rsync-client>)
2018/10/26 14:40:30 [4228] building file list
2018/10/26 14:40:35 [4228] rsync: writefd_unbuffered failed to write 4 bytes to socket [sender]: Connection timed out (110)
2018/10/26 14:40:35 [4228] rsync error: error in rsync protocol data stream (code 12) at io.c(1525) [sender=3.0.6]
```

#### rsync客户端：strace排查

通过strace来跟踪rsync进程执行时的系统调用和所接收的信号，但未发现系统级明显异常：

```
lstat("<rsync-test-pkg>-SNAPSHOT.jar", 0x7fff7d6e0000) = -1 ENOENT (No such file or directory)
select(5, [4], [3], [3], {30, 0})       = 2 (in [4], out [3], left {29, 999998})
select(5, [4], [], NULL, {30, 0})       = 1 (in [4], left {29, 999999})
read(4, "\0\0\0\34", 8184)              = 4
write(3, "\26\0\0\7\1\10\0\3\0\240\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0", 26) = 26
select(5, [4], [], NULL, {30, 0}./
<rsync-test-pkg>-SNAPSHOT.jar
)       = 0 (Timeout)8.79MB/s    0:00:02
select(5, [4], [], NULL, {30, 0})       = 0 (Timeout)
select(5, [4], [], NULL, {30, 0}
[receiver] io timeout after 60 seconds -- exiting
rsync error: timeout in data send/receive (code 30) at io.c(140) [receiver=3.0.9]
)       = 1 (in [4], left {28, 559609})
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=15649, si_status=30, si_utime=20, si_stime=5} ---
wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 30}], WNOHANG, NULL) = 15649
wait4(-1, 0x7fff7d6e11e4, WNOHANG, NULL) = -1 ECHILD (No child processes)
rt_sigreturn()                          = 1
read(4, "", 8184)                       = 0
write(2, "rsync: connection unexpectedly c"..., 77rsync: connection unexpectedly closed (115 bytes received so far) [generator]) = 77
write(2, "\n", 1
)                       = 1
rt_sigaction(SIGUSR1, {SIG_IGN, [], SA_RESTORER, 0x7f1e73cca670}, NULL, 8) = 0
rt_sigaction(SIGUSR2, {SIG_IGN, [], SA_RESTORER, 0x7f1e73cca670}, NULL, 8) = 0
getpid()                                = 15648
kill(15649, SIGUSR1)                    = -1 ESRCH (No such process)
write(2, "rsync error: error in rsync prot"..., 89rsync error: error in rsync protocol data stream (code 12) at io.c(605) [generator=3.0.9]) = 89
write(2, "\n", 1
)                       = 1
exit_group(12)                          = ?
+++ exited with 12 +++
```

#### 问题根源：网络质量

由于发布系统与qa环境不在同一个机房，存在跨机房调用的情况，而两端间的链路是走的公网VPN线路，确实可能存在质量不稳定的情况。

在执行rsync命令异常的qa环境服务器上，抓到的包是这样的：

![img](/img/in-post/post-181026-rsync-error-timeout/WechatIMG2288.jpeg)

而我们在与发布系统在同一机房网络环境下，抓到的包是这样的：

![img](/img/in-post/post-181026-rsync-error-timeout/WechatIMG2289.jpeg)

很明显，跨机房网络传输数据包中充斥着丢包、乱序以及重传，从而导致传输失败；

而在稳定的网络环境下，tcp包有序进行，rsync传输毫无问题。

#### TCP协议

这边在相同的源服务器上部署了nginx，通过80端口，开放http协议，供相同的客户端进行下载，结果发现wget相同文件，没有任何问题，而且网速还挺快的：

```
# wget http://<sitename>/<rsync-test-pkg>-SNAPSHOT.jar
--2018-10-26 10:57:45--  http://<sitename>/<rsync-test-pkg>-SNAPSHOT.jar
Resolving <sitename> (<sitename>)... 10.20.51.127
Connecting to <sitename> (<sitename>)|10.20.51.127|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 60606801 (58M) [application/java-archive]
Saving to: ‘<rsync-test-pkg>-SNAPSHOT.jar’

100%[==================================================================================================================================================================>] 60,606,801  9.81MB/s   in 5.2s   

2018-10-26 10:57:50 (11.1 MB/s) - ‘<rsync-test-pkg>-SNAPSHOT.jar’ saved [60606801/60606801]
```

但对其流量抓包，发现仍然有丢包的现象：

![img](/img/in-post/post-181026-rsync-error-timeout/WechatIMG2292.jpeg)

这就说明了一个现象：

rsync的网络传输协议特别依赖网络质量稳定性，推测当其tcp丢包、乱序或重传次数到达一定阈值时，便造成文件同步失败。

### 缓解

询问了网络的同事，没有办法快速解决，除非花钱买专线，但资源成本太高，为解决跨机房数据同步这个问题，小题大做了。

所以在目前网络环境下，无法从根源上有效解决该问题。

这边尝试过让异常用户重启下rsync客户端服务器，发现重启后能够同步成功，给到反馈发布失败的用户，也是给到该解决办法，即使用户很不理解为什么发布失败为什么需要他们重启服务器来解决，其实我也表示很无奈啊…

### 后记

当然，问题还是存在，rsync不行，这边就需要考虑通过其他方式来完成机房间文件存储同步了，这块需要后续调研下。

另外对于rsync在弱网环境下的服务质量，后续也需要看下rsync源码，查下问题究竟在哪里，同步失败的条件是什么。

[1]: http://chengkinhung.blogspot.com/2012/09/linuxrsyncdebug.html