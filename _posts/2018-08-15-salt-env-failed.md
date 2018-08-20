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

### 前言

上文[《salt执行带密码参数的rsync命令失败》][1]中提到，rsync使用加密方式同步文件，并且用salt执行命令行，必须将密码配到salt的环境变量中，我们后面的策略是在同步代码前，先执行一遍同步环境变量的命令，具体执行步骤如下：

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
<target>:
----------
          ID: environment_variables
    Function: environ.setenv
        Name: rsync
      Result: True
     Comment: Environ values were set
     Started: 15:31:09.216990
    Duration: 49.057 ms
     Changes:   
              ----------
              RSYNC_PASSWORD:
                  xxx
              USER:
                  root

Summary for <target>
------------
Succeeded: 1 (changed=1)
Failed:    0
------------
Total states run:     1
Total run time:  49.057 ms

# salt <target> environ.item '[USER, RSYNC_PASSWORD]'
<target>:
    ----------
    RSYNC_PASSWORD:
        xxx
    USER:
        root
```

以上方式能够解决rsync加密同步文件的问题。

### 背景

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

### 更新【08/20/2018】

由于SLS状态文件执行失败的问题不仅影响到了文件加密同步，同时影响其他状态文件执行，必须进行修复。

最后发现卸载salt-minion客户端、删除/etc/salt/*所有文件，重新安装salt-minion，问题可以得到解决。

于是全量扫了下服务器，将所有有问题的服务器全部列出，单独进行修复处理。

由于涉及到卸载操作，而且SLS本身处于不可用状态，所以没什么好办法，只能一台台SSH上搞，纯手工搬砖。

问题原因暂未找到，猜想应该是某位同学执行了一条与EPEL相关的状态命令，而且这条命令相当具有破坏性，只要一跑，除非重装salt，不然就废了。

当前能做的只有强化下salt-master服务器的权限，避免被未经授权的同学上去玩坏。

[1]: https://stephenzhou.net/2018/07/11/salt-failed-run-rsync-with-password/