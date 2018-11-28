---
layout: post
title: "权威域域名能否做CNAME解析？"
subtitle: "Don't CNAME @, ever."
date: 2018-11-28 10:40AM
catalog: true
tags:
    - 运维
    - CDN
    - DNS
---

### 背景

在提具体问题前，我们先讲下什么是：“短链接域名”。

我们知道，在PC端浏览器访问一个网站首页，URL地址栏很简单，敲域名首页就可以了。譬如我司首页是：“https://www.hujiang.com/”，只需要在浏览器中直接输入：“www.hujiang.com”，甚至是：“hujiang.com”就可以了，完全没什么问题。

当我在沪江网校学习，发现了白金卡这个产品，觉得很赞很实惠，想要安利给你看，这时候我就要把这个产品的URL地址（如：“https://class.hujiang.com/category/12628435324”）粘贴下来，再复制给你，你拿到这个地址后，可以复制、粘贴到浏览器地址栏，也没有什么问题。

但如果我司运营同学想要把这个产品以短信、微信形式推送到你，问题就来了：
- URL是不是太长了？一样是起到定位地址作用，能不能缩短点呢？
- 短信不像微信、邮件，是有紫薯限制的，URL占了这么多字位，其他的文案只能缩短，考验小编的语言功底；
- 虽说短信是按条收费的，但是谁也不想看到一条短信里密密麻麻全是字，内容精简才是王道！

我司有个短链接域名：hj.vc，专门用于URL跳转，间接实现了域名缩短的效果，如前面提到的长地址，我们现在用短地址（hj.vc/a/1）就能访问到啦！是不是很神奇？

#### 推广短信

![img](/img/in-post/post-181128-top-domain-cname/WechatIMG2464.png)

#### 点击短链接，转化后效果

![img](/img/in-post/post-181128-top-domain-cname/WechatIMG2465.png)

### 转化率问题

我司某部门有活动，用到短信短链接进行推广，运营同学反馈短信短链接的点击到目的页的转化数量，有较大幅度的衰减，也即用户触达率过低。

那么监控是如何设计的呢？在目的页的底部包含了监测元素，也就意味着：目的页需要在完全加载完后，监测元素才能正常执行，从而将用户到达目的页的状态计入计数器，最终用来统计用户触达率。

然而，当短链接跳转到目的页时，如果跳转期间页面加载时间较大的话，会造成监控状态失败，也即造成转化流失。

### 深入分析

通过上面的问题排查，我们推测出了几个可能会影响跳转期间，加载时间较大的因素：
- hj.vc这个域名未走CDN加速，用户请求直接回源站，可能会影响到小网络或偏远地区用户访问（国外用户访问不走CDN加速也会受影响，但短信推送的目标用户绝大部分都是国内手机用户，所以海外用户影响范围可控）；
- 短链接跳转到目的页的过程中，调用到了大数据部门的一个“中间域名”，中间域名的访问质量，同样也对转化的成功与否起到重要作用；
- 最后，目的页本身的响应情况，决定了最终能否成功“转化”。

直接需要运维这边跟进的是第一项hj.vc未上CDN的问题，第二、三项中的“中间域名”及目的页都是部署了CDN服务的，页面本身的加载情况由开发的同学负责，我们不做过多参与。

所以，我们的任务就是将短链接域名部署CDN服务。

### DNS解析问题

hj.vc在CDN侧的部署工作已完成，但是我们在DNS解析的时候遇到了问题：无法直接在DNS平台上添加权威域域名的CNAME记录，很明显这个操作是被DNS平台所禁止的。

那为什么会禁止呢？我们网上搜了下，其他DNS平台也会遇到类似问题，其得出的结论是：[CNAME与MX记录不能共存][1]，依据是[rfc-1034][2] Domain Concepts and Facilities中指出：

> From RFC1034 section 3.6.2: If a CNAME RR is present at a node, no other data should be present; this ensures that the data for a canonical name and its aliases cannot be different.

大致意思是：如果一个域名配置了CNAME，那么就不应该再配置其他类型的数据了，这保证了CNAME地址的一致性。

如果我执意要将CNAME和MX记录同时解析，是不是真的会出现问题呢？我们来尝试重现下就知道了。

### 异常重现

我在易名上购有一测试域名：“pingduodo.com”，这次总算是派上用处了！

我们在权威域域名上同时添加了CNAME以及MX记录（易名支持同时添加CNAME以及MX记录）：

![img](/img/in-post/post-181128-top-domain-cname/WechatIMG2469.png)

添加完毕后，我们进行dig测试：

#### 先解析CNAME，再解析MX

```
$ dig pingduodo.com

; <<>> DiG 9.10.6 <<>> pingduodo.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 26460
;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;pingduodo.com.			IN	A

;; ANSWER SECTION:
pingduodo.com.		60	IN	CNAME	hj.vc.wswebcdn.com.
hj.vc.wswebcdn.com.	60	IN	A	114.236.93.27
hj.vc.wswebcdn.com.	60	IN	A	58.220.40.132
hj.vc.wswebcdn.com.	60	IN	A	58.220.40.16

;; Query time: 188 msec
;; SERVER: 192.168.18.244#53(192.168.18.244)
;; WHEN: Wed Nov 28 10:32:52 CST 2018
;; MSG SIZE  rcvd: 108

$ dig pingduodo.com MX

; <<>> DiG 9.10.6 <<>> pingduodo.com MX
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 60022
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 0

;; QUESTION SECTION:
;pingduodo.com.			IN	MX

;; ANSWER SECTION:
pingduodo.com.		60	IN	CNAME	hj.vc.wswebcdn.com.

;; AUTHORITY SECTION:
wswebcdn.com.		60	IN	SOA	dns1.wswebcdn.org. webmaster.glb0.lxdns.com. 1422577239 10800 3600 604800 60

;; Query time: 51 msec
;; SERVER: 192.168.18.245#53(192.168.18.245)
;; WHEN: Wed Nov 28 10:33:28 CST 2018
;; MSG SIZE  rcvd: 134
```

#### 先解析MX，再解析CNAME

```
$ dig pingduodo.com MX

; <<>> DiG 9.10.6 <<>> pingduodo.com MX
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 52904
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;pingduodo.com.			IN	MX

;; ANSWER SECTION:
pingduodo.com.		600	IN	MX	1 mail.pingduodo.com.

;; Query time: 81 msec
;; SERVER: 192.168.18.244#53(192.168.18.244)
;; WHEN: Wed Nov 28 13:46:34 CST 2018
;; MSG SIZE  rcvd: 63

$ dig pingduodo.com

; <<>> DiG 9.10.6 <<>> pingduodo.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 44987
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;pingduodo.com.			IN	A

;; ANSWER SECTION:
pingduodo.com.		600	IN	CNAME	hj.vc.wswebcdn.com.
hj.vc.wswebcdn.com.	300	IN	A	150.138.170.21
hj.vc.wswebcdn.com.	300	IN	A	150.138.107.8

;; Query time: 121 msec
;; SERVER: 192.168.18.244#53(192.168.18.244)
;; WHEN: Wed Nov 28 13:46:38 CST 2018
;; MSG SIZE  rcvd: 103
```

我们发现：
- 在第一次先解析了CNAME地址后，MX记录被缓存住了结果，请求MX记录同样返回了CNAME记录，解析异常；
- 而第二次首先指定请求MX记录，再请求CNAME地址，结果不会被缓存，解析正常。

该结论匹配CloudXNS的测试结论：

> 递归DNS服务器在查询某个常规域名记录（非CNAME记录）时，如果在本地cache中已有该域名有对应的CNAME记录，则会开始用该别名记录来重启查询。

CloudXNS同时指出：

> 实际上除了CNAME和MX不能共存外，已经注册了CNAME类型的域名记录是不能再注册除DNSSEC相关类型记录（RRSIG、NSEC等）之外的任何其他类型记录（包括MX、A、NS等记录）。

也即：**只要配了SOA/NS/MX/A记录的域名，不能再配CNAME记录，否则可能影响到原SOA/NS/MX/A记录。**

另外，[StackExchange][3]上也有类似解释，特别是在权威域上进行该操作：

> Not possible - this would conflict with the SOA- and NS-records at the domain root.
> From RFC1912 section 2.4: "A CNAME record is not allowed to coexist with any other data."

> This is implementation specific and dangerous advice. Don't CNAME @, ever. 

由此，确认了权威域域名无法配置CNAME记录，即无法为权威域部署CDN服务。

我们发现，无论是BAT，还是谷歌、脸谱、推特，无一家对权威域域名进行CNAME解析，全为A记录。

当然，我们不排除大厂用到BGP技术，一个IP地址也可以做到全球覆盖(BGP AnyCast)，这里就不做延伸了。

### 缓解

由于hj.vc无法直接部署CDN服务，我们就需要考虑其他方法来实现了，最终方案如下：
- 使用a.hj.vc，但因没有*.hj.vc证书，所以暂时用不起来，需要等新证书下发后才能使用；
- 临时使用a.yeshj.com，既有证书，又可以部署CDN加速，只是需要开发同学改下调用接口。

最终采用第二个方案，并已部署至产线。

### 后记

使用DNS不能想当然，想配什么记录就配什么记录，必须按照协议规则使用，这块之前没研究过，正好乘着这次机会梳理下，避免乱用后踩雷。

[1]: https://www.cloudxns.net/Support/detail/id/130.html "【CloudXNS码农提示】为何CNAME和MX不能共存？"
[2]: https://tools.ietf.org/html/rfc1034 "DOMAIN NAMES - CONCEPTS AND FACILITIES"
[3]: https://serverfault.com/questions/430970/cname-for-top-of-domain "CNAME for top of domain?"