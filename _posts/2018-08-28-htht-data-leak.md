---
layout: post
title: "华住酒店用户数据泄露引发的思考"
subtitle: "还请珍视你的隐私！"
date: 2018-08-28 09:21PM
catalog: true
tags:
    - 安全
---

今天安全圈子又炸开了锅，华住酒店用户数据疑造泄露，号称涉及上亿条用户个人信息以及开房记录。

说时迟那时快，同时又爆出已有不法分子开始在所谓的“暗网”（非公开的互联网）上进行贩卖，相信用户数据已造泄露，究竟多少人拿到了这些数据，我们不得而知。

借着这个话题，我们今天来聊聊隐私与安全。

### 企业

用户成就了企业，没有一个个用户的积累，企业便无法做大做强。在企业逐渐扩张后，掌握的用户信息便越来越大，作为一个负责人的企业，无论是为自己（商业竞争方面），还是为他人（防止用户信息造泄露，伤害用户利益），都必须尽自己的全力，保护好这些数据。

这边先拿脸书开刷，今年3月中旬，相信大家应该还记得，Facebook公司因不恰当的收集用户隐私数据（这些数据在“我们”看来并非高危数据，只不过是一些好友列表及点赞数据），并被第三方分析机构所利用，由此掀起轩然大波，受此影响，脸书公司股价大跌，脸书CEO扎克伯格也因此身价大大缩水。

此次泄露事件的源头是某用户在Github（全世界最大的开源社区）上传了华住酒店某应用软件的程序代码，代码中包含了数据库敏感信息，一些不怀好意的黑客发现了这个漏洞，乘机把数据库内的所有内容全部下载了下来，也就是我们所谓的“脱裤”，涉及到的数据则是赤裸裸的用户姓名、手机号、身份证号、登录密码以及酒店订单记录等重要信息，比起脸书所谓泄露的“隐私数据”严重等级不知道高了多少。

就事论事，我们提出以下几点疑问：
- 作为企业内部的软件代码，为什么要上传到开源平台上去？
- 就算企业已打算将程序开源，为什么不将敏感数据进行数据脱敏？
- 数据库IP及端口为什么要对公网开放？
- 企业安全部门的作用何在？如此严重的泄漏事件为何没有及时发现？亦或是压根就没有安全部门？
- 企业员工安全意识在哪里？企业的安全规章制度在哪里？

无论最终原因是什么，企业主体难逃其咎，必须要为此事负责。

### 用户

李彦宏说过，国人对于隐私问题比较开放，或者说没那么敏感，如果通过交换隐私而获得便捷、效率、安全，在很多情况下他们是愿意这么做的。

个人觉得挺有道理的，怎么理解他这段话呢？我们随便举个几个例子：
- 注册滴滴，送打车券，你打不打？
- 注册美团，送盒水果，你吃不吃？
- 注册摩拜，共享单车免费骑，你骑不骑？

我们一边沉溺于薅羊毛的快感中，一边吐槽铺天盖地的垃圾电话和短信，有没有想过，也可能是因为自己对隐私数据管理的不够重视，才导致了现状的发生？

我们再看下隔壁日本，日本人对待隐私问题也有他们自己的执着，我们可以了解一下：
- 没有身份证，办事使用驾照或印章，没有谁能随随便便冒充谁
- 没有移动支付，只用信用卡，电话号码不会被恶意泄露
- 履历档案中只要求姓名、出生年月、教育情况、工作履历等客观事实，家庭成员关系、民族、政治面貌、宗教信仰等信息属个人隐私，一般不会写出来

对比我们，的确在对待隐私方面，更是多了一份严谨的态度在其中。

我们既然已经生活在Hard模式中，应该多留意一些心眼，将自己的隐私数据保护起来，譬如：
- 不随意登录、注册来路不明的网站
- 定期修改密码，并注意保存
- 数据脱敏，发朋友圈时涉及姓名、身份证号或订单号等信息时，注意打码，或者索性不要发
- 无论在哪里，留意摄像头，盯防背后的眼睛

### 三方监管机构

仅仅依靠企业或用户自律肯定是不够的，必须同时依靠外部力量（包括政府或第三方机构）进行监管或是制约，才能更好的保证隐私数据不被泄露。

对由于自身原因导致泄露用户数据的企业主体，相应的处罚措施能够跟上，罚到企业清楚不顾安全的风险有多大后，各企业对安全方面的投入也会更大，数据泄露的风险也会随之降低。

同时各方应努力提高漏洞监测、扫描机制，也可将泄密风险控制在最小范围之内。对于及时检测出泄露风险问题的企业或个人，给予一定的奖励，让好人不再吃亏，也是一个不错的办法。

还有一点就是如何对数字虚拟货币交易进行监管，只要留有漏洞，卖家就有可乘之机。此次暗网卖家要求通过比特币或门罗币进行交易，类似去年4月爆发的勒索病毒，也是索要比特币，试想为什么不是提供支付宝或微信二维码支付赎金呢？聪明的你一定已经明白了，对吧？

### 后记

你的隐私可能早已不是你的隐私，但还请珍视你的隐私，因为你有你自己重视了，别人才会重视。

裸奔有风险，操作需谨慎！