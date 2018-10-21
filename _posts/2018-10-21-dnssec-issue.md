---
layout: post
title: "DNSSEC配置导致BIND服务宕机"
subtitle: "又又踩到一坑！"
date: 2018-10-21 05:30PM
catalog: true
tags:
    - 运维
    - DNS
---

### 背景

又是周末，又是DNS问题……

开发同学在后端群里保障，QA环境域名解析失败，立即上线处理。

### 问题

可以复现，主备两台都失败，结果如下，可以看到解析状态均为：`SERVFAIL`：

```
# dig www.baidu.com @<qa_dns_1>

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.47.rc1.el6_8.4 <<>> www.baidu.com @<qa_dns_1>
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 37137
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;www.baidu.com.			IN	A

;; Query time: 35 msec
;; SERVER: <qa_dns_1>#53(<qa_dns_1>)
;; WHEN: Sun Oct 21 16:22:06 2018
;; MSG SIZE  rcvd: 31

# dig www.baidu.com @<qa_dns_2>

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.47.rc1.el6_8.4 <<>> www.baidu.com @<qa_dns_2>
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 50320
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;www.baidu.com.			IN	A

;; Query time: 28 msec
;; SERVER: <qa_dns_2>#53(<qa_dns_2>)
;; WHEN: Sun Oct 21 16:22:08 2018
;; MSG SIZE  rcvd: 31
```

cpu、内存、磁盘空间均正常，查看系统日志，发现问题：

```
# tail /var/log/messages
Oct 21 16:20:39 <hostname> named[9853]:   validating @0x7eff6420ecf0: com DS: bad cache hit (./DNSKEY)
Oct 21 16:20:39 <hostname> named[9853]: error (broken trust chain) resolving 'www.baidu.com/AAAA/IN': 119.75.219.82#53
```

查询第一条出问题的记录，发现早上11点已经有报错了：

```
# 第一条日志，解析soa失败
Oct 21 11:01:07 <hostname> named[9853]:   validating @0x7eff5c0a81d0: com SOA: got insecure response; parent indicates it should be secure
Oct 21 11:01:07 <hostname> named[9853]: error (no valid RRSIG) resolving 'cctalk.com/DS/IN': 192.35.51.30#53

# DNSKEY不受信任
Oct 21 11:01:14 <hostname> named[9853]: validating @0x7eff6c0b6930: . DNSKEY: unable to find a DNSKEY which verifies the DNSKEY RRset and also matches a trusted key for '.'
Oct 21 11:01:14 <hostname> named[9853]: validating @0x7eff6c0b6930: . DNSKEY: please check the 'trusted-keys' for '.' in named.conf.
Oct 21 11:01:14 <hostname> named[9853]: error (no valid KEY) resolving './DNSKEY/IN': 192.5.5.241#53

# 解析失败
Oct 21 11:02:44 <hostname> named[9853]: error (broken trust chain) resolving './NS/IN': 192.5.5.241#53
Oct 21 11:02:44 <hostname> named[9853]: error (broken trust chain) resolving 'www.baidu.com/A/IN': 119.75.219.82#53
```

同时谷歌了下，发现[红帽论坛][1]上同样有类似问题讨论，结论是配置问题，而非BUG，也即：`NOTABUG`。

> It seems you have your named misconfigured. You have "dnssec-lookaside . trust-anchor dlv.isc.org.;" line in your options {} statement but you don't have the dlv.isc.org. DNSKEY in your named.conf. Basically you told named "try to validate all domains via ISC DLV key" but you didn't specify the key. Thus named failed to validate all domains. You can check this with the /usr/bin/dig utility

得出初步结论：原配置文件开启了DNS安全扩展（DNSSEC）参数，非权威DNS不能开启这个配置，否则会造成dns请求为不信任链，最终导致解析失败。

### 缓解

1. 解决问题优先，重启named服务。确实重启后恢复正常。

2. 为避免相同问题再次发生，修改BIND服务配置，关闭DNSSEC相关配置：
原BIND配置文件：
```
        dnssec-enable yes;
        dnssec-validation yes;

        /* Path to ISC DLV key */
        bindkeys-file "/etc/named.iscdlv.key";

        managed-keys-directory "/var/named/dynamic";
```
调整后的BIND配置文件：
```
        dnssec-enable no;
        dnssec-validation no;

        /* Path to ISC DLV key */
        bindkeys-file "/etc/named.iscdlv.key";

        managed-keys-directory "/var/named/dynamic";
```

3. 监控缺失，添加可用性监控，并配置微信报警。

### 后记

这两台DNS是很久之前我自己部署点，因为是测试环境DNS，非线上服务，所以用的都是默认配置，没有仔细对其进行调试，一些具体配置参数具体是干嘛的并没有深入研究，从而导致这次故障。

后面还是需要一步一个脚印，对自己的服务负责，无论是线上还是测试环境，都需要一视同仁。

[1]: https://bugzilla.redhat.com/show_bug.cgi?id=577639