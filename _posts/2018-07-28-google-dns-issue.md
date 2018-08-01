---
layout: post
title: "谷歌DNS问题排查记录"
subtitle: "DNS别老配8.8.8.8，坑着呢！"
date: 2018-07-28 10:44PM
catalog: true
tags:
    - 运维
    - DNS
    - CDN
---
### 背景
我司在菲律宾有个办公点，用户经常反馈使用CCTalk有问题，特别是晚高峰时间段，现象更为严重，通常排查下来，基本都是网络链路不稳定造成，那具体为何会不稳定呢？我们简单分析下：

由于跨国，各国ISP（网络运营商）均不同，中间还隔着个GFW，各区域的光缆带宽都有可能跑满，或者某处网络设备出现高负载或异常时，整条网络链路就会出现问题，或出现丢包，或出现延迟。

这个时候，为避免网络直连造成的网络抖动问题，客户端程序一般不能直接连接源站服务器，我们会用到CDN，在中间区域（譬如香港）做一层代理，以减少网络延迟及丢包，提高网络质量。

于是，CDN节点的稳定性也就直接影响到了客户端程序运行稳定性。

之前菲律宾没有专门用来做监控的服务器，遇到问题都通过当地IT反馈过来，沟通成本极高，只能头痛医头脚痛医脚，完全没办法深入排查，间接造成了客户天天投诉，运维好像又不管事的现象，讲道理，不是我们不管，真的是臣妾做不到啊…

因为前几天菲律宾办公室网络设备故障，导致了当地网络中断了几个小时，由于没有监控，总部领导也是过了几个小时通过用户投诉才知道问题。领导自然是不开心了，断了这么久网可是直接影响生意的啊。于是借着这个契机，终于给到了运维一台服务器，NAT了SSH和zabbix server的出口IP，这边总算是有了台服务器可以用来搞监控了。

想到再也不用靠IT问这问那来排查问题了，工作效率直线提升，心里倍儿爽！于是马上连了上去，看看网络到底是有多差，随后很快就发现问题了。

### 问题
从总部BI打点日志上能查到，客户端程序请求某个URL经常出现超时，这边尝试手工访问这个URL，看下返回情况，结果如下，大家也可以看看，好像哪里有点不对劲：

```
# curl -I https://mc.hujiang.com/widgets/widget?widgetKey=5075ba12085b4ea7a9f693b478d09301#/teacher/scene/select -A hjops
HTTP/1.1 200 OK
Date: Fri, 27 Jul 2018 14:20:19 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 4885
Connection: keep-alive
Vary: Accept-Encoding
Vary: Accept-Encoding
X-DNS-Prefetch-Control: off
X-Frame-Options: SAMEORIGIN
X-Download-Options: noopen
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
X-Server-ID: server host
X-UA-Compatible: IE=edge,chrome=1
X-Powered-By: Node
Vary: Accept-Encoding
ETag: "1315-FsAjVhOtF7meyFmhLPNWPy37jC0"
X-IN-APIGATEWAY: b7-84
X-IN-WAF: NH7-14/04
Server: API-GATEWAYSSL/1.0
X-IN-APIGATEWAYSSL: b7-129
X-Ser: BC170_dx-lt-yd-jiangsu-zhenjiang-3-cache-3, BC198_US-DistColumbia-washingtonDC-1-cache-1, BC116_HK-xianggang-xianggang-4-cache-2
X-Cache: MISS from BC116_HK-xianggang-xianggang-4-cache-2(baishan)

# curl -I https://mc.hujiang.com/widgets/widget?widgetKey=b426ba8cca494067b291c9b164844817#/teacher/scene/select -A hjops
HTTP/1.1 200 OK
Date: Fri, 27 Jul 2018 14:21:26 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 4885
X-DNS-Prefetch-Control: off
X-Frame-Options: SAMEORIGIN
X-Download-Options: noopen
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
X-Server-ID: server host
X-UA-Compatible: IE=edge,chrome=1
X-Powered-By: Node
ETag: "1315-s1gk8Ap7kzglB4a6adwYn9YoqUs"
X-IN-APIGATEWAY: b7-130
X-IN-WAF: NH7-14/04
Server: API-GATEWAYSSL/1.0
X-IN-APIGATEWAYSSL: b7-170
X-Via: 1.1 PSxgHK5hc39:7 (Cdn Cache Server V2.0), 1.1 lb17:2 (Cdn Cache Server V2.0)
Connection: keep-alive
X-Dscp-Value: 0
```

看起来好像长差不多，都是200 OK，但再仔细看看，怎么两个请求，Header参数不一样呢？我们具体来看：

- 第一条`X-Ser`里包含字段`baishan`，说明用到了白山云CDN节点；
- 第二条`X-Via`里包含字段`Cdn Cache Server V2.0`，说明用到了网宿CDN节点；

这两点透露了两个信息：
- 同样的请求，解析到了两个不同的CDN代理节点
- 同样的请求，解析到了两家不同的CDN供应商线路

造成的结果就是，客户端程序发起HTTPS请求，上一秒还是节点A响应的，虽然完成了一次TCP连接，但到了下一秒，客户端程序又发送给了节点B，虽然不清楚具体程序实现逻辑，但猜想A节点可能会保留一些客户端数据，突然又到了B节点，先前保留的数据找不到了，所以原本应该正常响应的请求，这会儿就歇菜了。

什么原因会造成响应的节点不一致呢？这个时候又得提到前面说过的DNS和CDN了，传统CDN一般都是根据用户的LocalDNS，调度分配给离用户最近（同ISP、同城或同省）的CDN节点（准确的说，应该是离用户所使用的LocalDNS最近的），以此来保证最优网络质量，这是CDN这层做的调度。另外，我厂较为特殊，权威DNS分别对菲律宾和美国做了区域解析，正常情况下，菲律宾的请求会被调度到网宿CDN，美国的请求会被调度到白山云CDN，怎么到这边就不灵光了呢？为啥菲律宾的请求，一会儿被调度到了网宿，一会儿又到了白山云呢？

以下是我们通过一个小脚本，循环跑了几次，以重现问题，脚本链接：[戳我][1]

```
# ./ckdnsresov.sh mc.hujiang.com 8.8.8.8
1
hjcdn-os-sec.hjdns.net.
overseacname.hujiang.com.baishan-cloud.net.
zhjiang.v.baishan-cloud.net.
107.155.25.120
107.155.25.117
107.155.25.118
107.155.25.116
+++
2
hjcdn-os-pri.hjdns.net.
public.hujia.102.cdn20.com.
220.243.239.52
+++
3
hjcdn-os-pri.hjdns.net.
public.hujia.102.cdn20.com.
220.243.239.52
+++
4
hjcdn-os-sec.hjdns.net.
overseacname.hujiang.com.baishan-cloud.net.
zhjiang.v.baishan-cloud.net.
107.155.25.120
107.155.25.117
107.155.25.118
107.155.25.116
+++
5
hjcdn-os-sec.hjdns.net.
overseacname.hujiang.com.baishan-cloud.net.
zhjiang.v.baishan-cloud.net.
107.155.25.117
107.155.25.118
107.155.25.116
107.155.25.120
+++
6
hjcdn-os-pri.hjdns.net.
public.hujia.102.cdn20.com.
220.243.239.52
+++
7
hjcdn-os-sec.hjdns.net.
overseacname.hujiang.com.baishan-cloud.net.
zhjiang.v.baishan-cloud.net.
107.155.25.120
107.155.25.117
107.155.25.118
107.155.25.116
+++
8
hjcdn-os-sec.hjdns.net.
overseacname.hujiang.com.baishan-cloud.net.
zhjiang.v.baishan-cloud.net.
107.155.25.120
107.155.25.117
107.155.25.118
107.155.25.116
+++
9
hjcdn-os-pri.hjdns.net.
public.hujia.102.cdn20.com.
220.243.239.52
+++
10
hjcdn-os-sec.hjdns.net.
overseacname.hujiang.com.baishan-cloud.net.
zhjiang.v.baishan-cloud.net.
107.155.25.120
107.155.25.117
107.155.25.118
107.155.25.116
+++
All done
```

看来罪魁祸首就是DNS，查了下这台机器配的DNS，配的是8.8.8.8，谷歌的DNS。

我们知道，[谷歌公共DNS][2]用的是[Anycast][3]，分布在各地的DNS都是BGP出口，IP使用的都是8.8.8.8和8.8.4.4，谷歌根据客户端的地理位置信息，递归查询后返回离客户端位置最近的解析结果。

### 缓解
问题原因稍后再做分析，先缓解问题，谷歌上搜了下菲律宾的公共DNS，随便试了两个都没问题，修改服务器DNS配置到这两个地址，问题解决。

![img](/img/in-post/post-google-dns-issue/ph-pub-dns.png)

### 原因
问题缓解后，我们又单独做了下8.8.8.8的DNS TRACE查询，却无法重现问题，每次TRACE的结果都是符合预期的：

```
# dig mc.hujiang.com @8.8.8.8 +trace

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.17.rc1.el6_4.6 <<>> mc.hujiang.com @8.8.8.8 +trace
;; global options: +cmd
.			103123	IN	NS	a.root-servers.net.
.			103123	IN	NS	b.root-servers.net.
.			103123	IN	NS	c.root-servers.net.
.			103123	IN	NS	d.root-servers.net.
.			103123	IN	NS	e.root-servers.net.
.			103123	IN	NS	f.root-servers.net.
.			103123	IN	NS	g.root-servers.net.
.			103123	IN	NS	h.root-servers.net.
.			103123	IN	NS	i.root-servers.net.
.			103123	IN	NS	j.root-servers.net.
.			103123	IN	NS	k.root-servers.net.
.			103123	IN	NS	l.root-servers.net.
.			103123	IN	NS	m.root-servers.net.
;; Received 228 bytes from 8.8.8.8#53(8.8.8.8) in 1877 ms

com.			172800	IN	NS	e.gtld-servers.net.
com.			172800	IN	NS	b.gtld-servers.net.
com.			172800	IN	NS	j.gtld-servers.net.
com.			172800	IN	NS	m.gtld-servers.net.
com.			172800	IN	NS	i.gtld-servers.net.
com.			172800	IN	NS	f.gtld-servers.net.
com.			172800	IN	NS	a.gtld-servers.net.
com.			172800	IN	NS	g.gtld-servers.net.
com.			172800	IN	NS	h.gtld-servers.net.
com.			172800	IN	NS	l.gtld-servers.net.
com.			172800	IN	NS	k.gtld-servers.net.
com.			172800	IN	NS	c.gtld-servers.net.
com.			172800	IN	NS	d.gtld-servers.net.
;; Received 508 bytes from 198.41.0.4#53(198.41.0.4) in 2136 ms

hujiang.com.		172800	IN	NS	ns1.hjdns.com.
hujiang.com.		172800	IN	NS	ns2.hjdns.com.
hujiang.com.		172800	IN	NS	ns3.hjdns.com.
hujiang.com.		172800	IN	NS	ns4.hjdns.com.
hujiang.com.		172800	IN	NS	ns5.hjdns.com.
hujiang.com.		172800	IN	NS	ns6.hjdns.com.
;; Received 242 bytes from 192.35.51.30#53(192.35.51.30) in 1016 ms

mc.hujiang.com.		200	IN	CNAME	hjcdn-os-pri.hjdns.net.
hjcdn-os-pri.hjdns.net.	200	IN	CNAME	public.hujia.102.cdn20.com.
;; Received 105 bytes from 180.153.38.202#53(180.153.38.202) in 397 ms
```

解析请求首先被我司权威DNS调度到了网宿CDN，然后再被网宿CDN调度到了某个指定的边缘POP(Edge Points of Presence)节点。

我们也尝试刷新了谷歌DNS缓存，但刷新后还是可以重现问题，所以也不像是缓存的问题：

![img](/img/in-post/post-google-dns-issue/flush-google-dns-cache.png)

这个问题的原因暂时无解，暂时猜想是BGP路由关系，客户端发起的DNS查询请求不知为何，被路由到了北美的DNS去了，具体还需要深入研究才能得出结论。

### 后记
请谨慎配置8.8.8.8，特别是像我司配置了国外区域解析的，只要你配8.8.8.8，分配给你的CDN节点就是非大陆的，这不，随便解析了个，返回了俩弯弯的节点，好端端的国内请求，绕了一大圈才回源，响应效果肯定有损耗…

```
$ dig mc.hujiang.com @8.8.8.8 +short
hjcdn-os-pri.hjdns.net.
public.hujia.102.cdn20.com.
61.221.181.252
210.61.180.159
```

所以如果是国内的用户，运营商提供的DNS如果不灵了，请不要第一时间改8.8.8.8，可以改成我们国内的公共DNS，如：114.114.114.114，腾讯的DNSPod DNS：119.29.29.29，或者阿里、百度的DNS，效果并不比谷歌的差。

[1]: https://gist.githubusercontent.com/zhoufwind/94be2542ef23cadbd503d2efb373d932/raw/1611c7851114b80147e78cfa24052a70679ee181/ckdnsresov.sh "ckdnsresov.sh"

[2]: https://developers.google.com/speed/public-dns/faq "Google Public DNS FAQ"

[3]: https://en.wikipedia.org/wiki/Anycast "Wikipedia entry"