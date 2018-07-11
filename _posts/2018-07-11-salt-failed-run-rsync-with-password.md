---
layout: post
title: "salt执行带密码参数的rsync命令失败"
subtitle: "Salt进程环境变量的坑"
date: 2018-07-11 23:58:00 +0800
catalog: true
tags:
    - 运维
    - Salt
    - Rsync
---
### 背景
Rsync非加密传输方式不安全，需要加密传输。

### 问题
加密传输后的rsync命令需要添加 `--password-file` 参数，本机执行没有问题，但通过salt调用失败，提示权限验证失败：

> @ERROR: auth failed on module test
> rsync error: error starting client-server protocol (code 5) at main.c(1648) [Receiver=3.1.2]

### 缓解
深入研究后，了解rsync加密时读取密码内容的方式：
- 从--password-file文件中读取密码内容
- 从环境变量中读取密码内容
无论哪种方式，都将赋值给系统变量，供rsync程序调用，而salt的工作方式要求必须手工指定环境变量后，rsync才能正常获取密码内容。

当发现这个问题的时候，第一直觉就是环境变量的问题，但当真正分析的时候并没有立即往这方面细想，还认为是salt的功能问题，分别在论坛上提了问题：
- Stack Overflow: rsync auth failed when executed by salt. [Link][1]
- Github: #[48517][2]

但后面仔细想了想，rsync和salt本身应该都没有问题，排查可能的问题后，只剩下环境变量的坑了。

### 后记
虽然当前基本功能可用，但还有个很严重的问题，salt文档上说，环境变量只对当前进程有效，若真是如此，salt-minion进程挂掉后，必须再对环境变量进行赋值，否则又将失败。

> Support for getting and setting the environment variables of the current salt process. [Link][3]

针对这个问题，后期还需要进行测试。

[1]: https://stackoverflow.com/questions/51249187/rsync-auth-failed-when-executed-by-salt

[2]: https://github.com/saltstack/salt/issues/48517

[3]: https://docs.saltstack.com/en/latest/ref/states/all/salt.states.environ.html