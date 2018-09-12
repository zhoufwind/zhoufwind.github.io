---
layout: post
title: "谷歌新版Chrome浏览器不再信任赛门铁克签发证书"
subtitle: "还在用赛门铁克签发的证书？是时候更新了！"
date: 2018-09-12 11:34AM
catalog: true
tags:
    - 运维
    - SSL
    - CDN
---

### 背景

赛门铁克当时仗着自己CA大厂的身份，随意签发证书，更危险的事，签发的这些证书中，有部分证书权限很大，被签发者拥有自签权限，给各浏览器带来了严重的安全风险。

谷歌早就发现了这个问题，认为赛门铁克的这些行为严重违反了Chrome浏览器的[根证书政策][1]，提出[正式反馈][2]后，仍然我行我素，不知悔改，最后越过了谷歌的底线，最终决定逐渐不再信任赛门铁克签发的证书，[邮件链接][3]。

### 影响

当前，谷歌浏览器探如果测到网站仍使用赛门铁克签发的证书，会在Console中显示异常信息，截图如下：

![img](/img/in-post/post-180912-distrust-of-symantec-pki/WechatIMG2111.png)

Console提示信息如下：

> The SSL certificate used to load resources from https://gitlab.yeshj.com will be distrusted in M70. Once distrusted, users will be prevented from loading these resources. See https://g.co/chrome/symantecpkicerts for more information.

而根据[谷歌公告][4]，M70将在2018年9月13日发布测试版本，10月16日发布正式版本：

![img](/img/in-post/post-180912-distrust-of-symantec-pki/WechatIMG2113.png)

**如果网站证书在10月16日之前仍未更新赛门铁克签发证书的话，用户继续使用Chrome访问这些站点便会出现证书错误提示，除非换其他浏览器进行访问。**

如果用户当前使用Chrome 66版本，并访问在2016年6月1日前由赛门铁克签发证书的网站，浏览器会直接提示证书异常错误，必须更新谷歌浏览器版本：

![img](/img/in-post/post-180912-distrust-of-symantec-pki/WechatIMG2109.png)

### 问题

我司在今年3月就完成了所有证书的申请工作（选择靠谱的SSL供应商，避免出现类似问题），同时在所有网关设备上进行更新，包括源站出口网关，以及各家CDN节点证书，更新后这些报错信息便会消失。

但因为个别CDN供应商的证书管理平台上保存了多份域名证书，老证书没有及时的进行删除，CDN技术人员或业务上的同学在后台配置新域名时，有时候会不小心选择老证书，最终导致又出现不兼容证书的问题。

由于临近Chrome M70版本上线时间，这个问题必须赶在上线前解决，保证用户正常访问，从而避免线上故障。

### 缓解

由于公司域名数量较多，而且几乎每个域名都用了多家CDN，所以靠人工一个个比对哪个域名证书有问题，这种做法几乎是不可能的，必须自动化批量查找。

我司运维监控的伙伴很早之前就开始进行证书批量检查的工作，最早是批量查询各域名的SSL证书的过期时间，保证证书不会过期。若是监测到证书即将过期，立即发起报警，截图如下：

![img](/img/in-post/post-180912-distrust-of-symantec-pki/WechatIMG2107.png)

而这次稍许调整了下需求，改为批量查询各域名的SN序列号，并与最新证书的SN进行比对，若不一致，就把域名打印出来，这样所有配置有问题证书的域名，以及配置在源站或是CDN网关，便一目了然了。

拿到域名列表后，便可以开始证书替换工作了。

这边需要给运维监控同学点个赞，多亏了你们的自动化检测脚本，才能让我们及时发现问题，不至于在检测方面花费太多时间。

### 后记

谷歌这么做，虽然对各网站运营人员来说有些折腾，但其初衷却是好的，无非是想维护一个健康的互联网环境。在这个大前提下，跟着一起折腾几下也无可厚非。

随手搜了下，发现也有几家大厂面临同样问题，是时候更新了！

![img](/img/in-post/post-180912-distrust-of-symantec-pki/WechatIMG2103.png)

[1]: https://www.chromium.org/Home/chromium-security/root-ca-policy "Root Certificate Policy"
[2]: https://security.googleblog.com/2015/10/sustaining-digital-certificate-security.html "Sustaining Digital Certificate Security"
[3]: https://groups.google.com/a/chromium.org/forum/#!msg/blink-dev/eUAKwjihhBs/rpxMXjZHCQAJ "Intent to Deprecate and Remove: Trust in existing Symantec-issued Certificates"
[4]: https://security.googleblog.com/2018/03/distrust-of-symantec-pki-immediate.html "Distrust of the Symantec PKI: Immediate action needed by site operators"