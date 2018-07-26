---
layout: post
title: "CDN性能优化实例——回源HTTPS证书卸载"
subtitle: "减少源站SSL交互，减少源站压力"
date: 2018-07-27 01:39AM
catalog: true
tags:
    - 运维
    - CDN
---
### 案例一
公司某业务线有一域名：vocablist.hjapi.com，用来上传用户数据的，大多为POST请求，[之前文章][1]中也有提到过，POST请求比GET更耗时，更为依赖CDN回源链路质量。涉及到POST请求的域名，5XX状态码数量明显比其他GET请求的域名要来的更多。

这边特别用了两家CDN供应商进行测试，两家CDN表现一致，说明CDN侧的问题较小，很有可能与源站有关。

我们发现，其中有一家CDN特别良心，把5XX的原因分别列了出来：
![img](/img/in-post/post-cdn-origin-prot/cdn-origin-504.png)

其中大部分504错误都是由：“CDN连接源站失败(SSL)”这个异常贡献的，于是便猜测由于CDN与源站SSL层建连失败导致。

想要证实倒也简单，修改下域名的CDN配置，回源协议由原来的HTTPS改为HTTP即可。

修改后果不其然，5XX数量立马下降，基本可以确定与HTTPS回源有关。

状态码200响应情况：
![img](/img/in-post/post-cdn-origin-prot/cdn-sc-vocablist-2xx.png)

状态码5XX响应情况：
![img](/img/in-post/post-cdn-origin-prot/cdn-sc-vocablist-5xx.png)

### 案例二

正好今晚业务侧又有反馈，有一域名：ccbook.hjapi.com，用于微信小程序，超过10s服务端若未返回，微信判定请求失败，会断开与服务端的连接。

CDN的同学帮忙实时分析日志，确实发现客户端请求超过10s便主动断开了，此时CDN返回499状态码，499含义如下：

> 499, client has closed connection

业务侧开发同学反馈程序并不是长连接，源站响应很快，源站日志都不超过1s；

由上面的信息，可以推测：源站在1s不到的时间便完成响应处理，客户端（微信小程序）发起请求到源站，若超过10s未返回，客户端便断开连接，请求失败。

那么，这10s是花在哪步上呢？DNS解析？TCP建连？TLS建连？建连后传数据所消耗？

感觉问题又是出在源站网关SSL建连这块，要求业务方运维修改回源协议，由HTTPS调整为HTTP，调整完后再无异常发生，问题解决。

状态码499响应情况：
![img](/img/in-post/post-cdn-origin-prot/cdn-sc-ccbook-499.png)

回源504响应情况：
![img](/img/in-post/post-cdn-origin-prot/cdn-sc-ccbook-504.png)

### 问题

我们现在只能推测是源站网关SSL层建连失败的问题，后续需要查下网关SSL服务器日志，机器性能等参数，才能给到根本原因，但大方向应该是不会错了。

### 后记

今天这个发现一次性解决了两个问题，一是POST请求会出现大量5XX的问题，一是CDN返回大量499的原因，还算不错。

另外看了下5XX状态码，请求量相对较大的域名除了SSL建连失败导致5XX只是一类，还有其他类型的5XX错误，如:
- CDN错误
- 源站错误
- CDN连接源站失败
- CDN连接源站读超时
感觉坑还不少，后面还得一个个慢慢填。

[1]: https://stephenzhou.net/2018/06/27/why-post-expensive-then-get/ "为什么POST比GET更耗时？"