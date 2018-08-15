---
layout: post
title: "salt state.apply执行失败问题跟踪"
subtitle: "salt env配置的两种途径"
date: 2018-08-15 03:58PM
catalog: true
tags:
    - 运维
    - Salt
    - Rsync
---

### 背景

上文[《salt执行带密码参数的rsync命令失败》][1]中提到，rsync使用加密方式同步文件，并且用salt执行命令行，必须将密码配到salt的环境变量中，我们后面的策略是在同步代码前，先执行一遍同步环境变量的命令，具体如下：

```
# cat /path/to/salt/set_rsync_env.sls 
environment_variables:
  environ.setenv:
    - name: rsync
    - update_minion: True
    - value:
        USER: "root"
        RSYNC_PASSWORD: "xxx"

# salt <target> state.apply /path/to/salt/set_rsync_env
```

以上方式能够解决rsync加密同步文件的问题。

今天突然接用户反馈，rsync同步文件时出现异常，提示rsync auth验证失败，截图如下：

![img](/img/in-post/post-180815-salt-env-failed/walle-rsync-auth-failed.jpg)

### 问题

通过报错信息来看，基本确定是密码没存到环境变量中，于是手工执行了下：

```
# salt <target> environ.item '[USER, RSYNC_PASSWORD]' -l quiet
<target>:
    ----------
    RSYNC_PASSWORD:
    USER:
        root

# salt <target> state.apply /path/to/salt/set_rsync_env
<target>:
    Data failed to compile:
----------
    Pillar failed to render with the following messages:
----------
    Rendering SLS 'epel' failed. Please see master log for details.
ERROR: Minions returned with non-zero exit code
```

发现可以重现问题，但找来找去并未发现`epel`这个SLS状态文件，而且并不是所有机器都有问题，似乎像是salt-minion端缓存的问题，但清了缓存、重启salt-minion后仍然无法解决。

### 缓解

查了许久，还是没有思路，想下其他办法，先绕过这个错，让用户正常可用为先。

这时想到配置salt环境变量有两种方式，一种是我们已经用的`state.apply`这种状态文件方式，还有一种命令行`environ.setenv`这种方式直传，试了下，可行：

```
# salt <target> environ.setenv '{USER: "root", RSYNC_PASSWORD: "xxx"}' update_minion=True
# salt <target> environ.item '[USER, RSYNC_PASSWORD]' -l quiet
<target>:
    ----------
    RSYNC_PASSWORD:
        xxx
    USER:
        root
```

### 后记

曲线救国，先保证服务可用，具体故障原因，后面抽空再做分析。

[1]: https://stephenzhou.net/2018/07/11/salt-failed-run-rsync-with-password/