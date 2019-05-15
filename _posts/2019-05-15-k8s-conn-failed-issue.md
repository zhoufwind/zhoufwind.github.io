---
layout: post
title: "访问k8s服务连接失败问题排查"
date: 2019-05-15 01:33PM
catalog: true
tags:
    - 运维
    - K8S
    - PALO
    - TCP/IP
---

### 背景 & 问题

K8S部署了palo服务，在导数据时候频繁出现连接失败导问题，如下日志，在进行curl导入数据时，分别出现了如下3种异常：

- Connection timed out
- Connection refused
- No route to host

```
# curl --location-trusted -u test:test -T table1_data http://<NODE_IP>:28030/api/example_db/table1/_load?label=table1_2019051401
curl: (7) Failed connect to 192.152.13.194:8040; Connection timed out

# curl --location-trusted -u test:test -T table1_data http://<NODE_IP>:28030/api/example_db/table1/_load?label=table1_2019051401
curl: (7) Failed connect to <NODE_IP>:28030; Connection refused

# curl --location-trusted -u test:test -T table1_data http://<NODE_IP>:28030/api/example_db/table1/_load?label=table1_2019051401
curl: (7) Failed connect to 192.152.203.135:8040; No route to host
```

#### 正常的tcpdump包

注：tcpdump中flags标志位含义：
- [S] 表示这是一个SYN请求；
- [S.] 表示这是一个SYN+ACK确认包；
- [.] 表示这是一个ACT确认包；
- [P.] 表示这个是一个数据推送；
- [F.] 表示这是一个FIN请求；

client

```
1. 13:52:20.908197 IP 192.168.35.201.54111 > 192.168.36.145.28030: Flags [S], seq 590982416, win 29200, options [mss 1460,sackOK,TS val 1911533260 ecr 0,nop,wscale 9], length 0 # SYN J=590982416
3. 13:52:20.908478 IP 192.168.35.201.54111 > 192.168.36.145.28030: Flags [.], ack 872829976, win 58, options [nop,nop,TS val 1911533260 ecr 2447443106], length 0 # ACK K+1=872829976

4. 13:52:20.908567 IP 192.168.35.201.54111 > 192.168.36.145.28030: Flags [P.], seq 0:84, ack 1, win 58, options [nop,nop,TS val 1911533260 ecr 2447443106], length 84
7. 13:52:20.909969 IP 192.168.35.201.54111 > 192.168.36.145.28030: Flags [.], ack 184, win 60, options [nop,nop,TS val 1911533262 ecr 2447443108], length 0

8. 13:52:20.910058 IP 192.168.35.201.54111 > 192.168.36.145.28030: Flags [F.], seq 84, ack 184, win 60, options [nop,nop,TS val 1911533262 ecr 2447443108], length 0
10. 13:52:20.910282 IP 192.168.35.201.54111 > 192.168.36.145.28030: Flags [.], ack 185, win 60, options [nop,nop,TS val 1911533262 ecr 2447443108], length 0
```

Server
```
2. 13:52:20.910648 IP 192.168.36.145.28030 > 192.168.35.201.54111: Flags [S.], seq 872829975, ack 590982417, win 43440, options [mss 1460,sackOK,TS val 2447443106 ecr 1911533260,nop,wscale 9], length 0 # ACK J+1=590982417, SYN K=872829975

5. 13:52:20.910995 IP 192.168.36.145.28030 > 192.168.35.201.54111: Flags [.], ack 85, win 85, options [nop,nop,TS val 2447443107 ecr 1911533260], length 0
6. 13:52:20.912178 IP 192.168.36.145.28030 > 192.168.35.201.54111: Flags [P.], seq 1:184, ack 85, win 85, options [nop,nop,TS val 2447443108 ecr 1911533260], length 183

9. 13:52:20.912505 IP 192.168.36.145.28030 > 192.168.35.201.54111: Flags [F.], seq 184, ack 86, win 85, options [nop,nop,TS val 2447443108 ecr 1911533262], length 0
```

Client+Server合并：
```
# client发起3次握手
1. 14:07:20.942535 IP 192.168.35.201.55621 > 192.168.36.145.28030: Flags [S], seq 2782243378, win 29200, options [mss 1460,sackOK,TS val 1912433290 ecr 0,nop,wscale 9], length 0 # SYN J=2782243378
2. 14:07:20.942593 IP 192.168.36.145.28030 > 192.168.35.201.55621: Flags [S.], seq 728405943, ack 2782243379, win 43440, options [mss 1460,sackOK,TS val 2448343138 ecr 1912433290,nop,wscale 9], length 0 # ACK J+1=2782243379, SYN K=728405943
3. 14:07:20.942776 IP 192.168.35.201.55621 > 192.168.36.145.28030: Flags [.], ack 1, win 58, options [nop,nop,TS val 1912433290 ecr 2448343138], length 0 # ACK K+1

# client发送请求内容
4. 14:07:20.942872 IP 192.168.35.201.55621 > 192.168.36.145.28030: Flags [P.], seq 1:85, ack 1, win 58, options [nop,nop,TS val 1912433291 ecr 2448343138], length 84
# server ack
5. 14:07:20.942893 IP 192.168.36.145.28030 > 192.168.35.201.55621: Flags [.], ack 85, win 85, options [nop,nop,TS val 2448343138 ecr 1912433291], length 0
# server发送响应内容
6. 14:07:20.943969 IP 192.168.36.145.28030 > 192.168.35.201.55621: Flags [P.], seq 1:184, ack 85, win 85, options [nop,nop,TS val 2448343140 ecr 1912433291], length 183
# client ack
7. 14:07:20.944156 IP 192.168.35.201.55621 > 192.168.36.145.28030: Flags [.], ack 184, win 60, options [nop,nop,TS val 1912433292 ecr 2448343140], length 0

# client发起结束请求
8. 14:07:20.944240 IP 192.168.35.201.55621 > 192.168.36.145.28030: Flags [F.], seq 85, ack 184, win 60, options [nop,nop,TS val 1912433292 ecr 2448343140], length 0
# server发起结束请求
9. 14:07:20.944309 IP 192.168.36.145.28030 > 192.168.35.201.55621: Flags [F.], seq 184, ack 86, win 85, options [nop,nop,TS val 2448343140 ecr 1912433292], length 0
# client ack
10. 14:07:20.944494 IP 192.168.35.201.55621 > 192.168.36.145.28030: Flags [.], ack 185, win 60, options [nop,nop,TS val 1912433292 ecr 2448343140], length 0
```

#### 异常包

异常问题：
- 服务端收到客户端的请求，但未建立握手（无ack），而是服务端回包`seq=0`；
- 结束的时候并非正常的client发起FIN包`[F.]`，而是server发起RST包`[R.]`；

```
14:11:50.715990 IP 192.168.35.201.56075 > 192.168.36.145.28030: Flags [S], seq 761737745, win 29200, options [mss 1460,sackOK,TS val 1912703068 ecr 0,nop,wscale 9], length 0
14:11:50.716503 IP 192.168.36.145.28030 > 192.168.35.201.56075: Flags [R.], seq 0, ack 761737746, win 0, length 0
```

### 缓解

后检查发现由于yaml配置文件中，label配置有冲突，FE与BE的label都叫palo，只是单独添加role进行区分，但在`service`的`selector`配置中，并未指定是FE还是BE，所以导致服务端口混淆，明明请求的是FE，但访问到了BE的service去了。

将所有role去除，app改名为`palo-fe`及`palo-be`，进行显式区分，问题得到缓解。