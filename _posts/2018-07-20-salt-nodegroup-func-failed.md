---
layout: post
title: "由salt-master版本升级导致salt分组推送目标失败的问题分析"
subtitle: "不能完美向下兼容一定是有何难言之隐？"
date: 2018-07-20 12:22AM
catalog: true
tags:
    - 运维
    - Salt
---
### 背景
微信群中收到反馈，有同学通过配置管理平台（公司自研）批量推送命令模块失败，失败场景：只有选择“全部”，批量推送会失败，单独选择IP进行推送没有问题。

### 问题
开始以为是不是模块有问题，后来对该分组进行了下test.ping，发现同样失败，但单独test.ping分组中的某台机器缺返回正常，这样就说明应该是salt分组推送出现了问题。

为了缩小排查范围，单独对只有两台服务器的分组：“ELK”进行了测试：
```
# salt -N ELK test.ping
[DEBUG   ] Configuration file path: /etc/salt/master
[WARNING ] Insecure logging configuration detected! Sensitive data may be logged.
[DEBUG   ] Reading configuration from /etc/salt/master
[DEBUG   ] Including configuration from '/etc/salt/master.d/api.conf'
[DEBUG   ] Reading configuration from /etc/salt/master.d/api.conf
[DEBUG   ] Including configuration from '/etc/salt/master.d/eauth.conf'
[DEBUG   ] Reading configuration from /etc/salt/master.d/eauth.conf
[DEBUG   ] Including configuration from '/etc/salt/master.d/nodegroups.conf'
[DEBUG   ] Reading configuration from /etc/salt/master.d/nodegroups.conf
[DEBUG   ] Using cached minion ID from /etc/salt/minion_id: <salt-master_ip>
[DEBUG   ] Missing configuration file: /root/.saltrc
[DEBUG   ] MasterEvent PUB socket URI: /var/run/salt/master/master_event_pub.ipc
[DEBUG   ] MasterEvent PULL socket URI: /var/run/salt/master/master_event_pull.ipc
[DEBUG   ] nodegroup_comp(ELK) => [u'(', u'L@<es1_ip>,<es2_ip>,', u')']
[DEBUG   ] No nested nodegroups detected. Using original nodegroup definition: L@<es1_ip>,<es2_ip>,
[DEBUG   ] Initializing new AsyncZeroMQReqChannel for (u'/etc/salt/pki/master', u'<salt-master_ip>_master', u'tcp://127.0.0.1:4506', u'clear')
[DEBUG   ] Connecting the Minion to the Master URI (for the return server): tcp://127.0.0.1:4506
[DEBUG   ] Trying to connect to: tcp://127.0.0.1:4506
[DEBUG   ] Initializing new IPCClient for path: /var/run/salt/master/master_event_pub.ipc
[DEBUG   ] LazyLoaded local_cache.get_load
[DEBUG   ] Reading minion list from /var/cache/salt/master/jobs/41/8faba3d8956abb522f3d8958f20d9d24657e8d5512c3ac05c92a18fa513a48/.minions.p
[DEBUG   ] get_iter_returns for jid 20180719155834031057 sent to set([]) will timeout at 16:00:34.062570
[DEBUG   ] Checking whether jid 20180719155834031057 is still running
[DEBUG   ] Initializing new AsyncZeroMQReqChannel for (u'/etc/salt/pki/master', u'<salt-master_ip>_master', u'tcp://127.0.0.1:4506', u'clear')
[DEBUG   ] Connecting the Minion to the Master URI (for the return server): tcp://127.0.0.1:4506
[DEBUG   ] Trying to connect to: tcp://127.0.0.1:4506
[DEBUG   ] Passing on saltutil error. Key 'u'retcode' missing from client return. This may be an error in the client.
[DEBUG   ] Passing on saltutil error. Key 'u'retcode' missing from client return. This may be an error in the client.
[DEBUG   ] Passing on saltutil error. Key 'u'retcode' missing from client return. This may be an error in the client.
[DEBUG   ] return event: {u'<es1_ip>': {u'failed': True}}
[DEBUG   ] LazyLoaded localfs.init_kwargs
[DEBUG   ] LazyLoaded localfs.init_kwargs
[DEBUG   ] LazyLoaded no_return.output
<es1_ip>:
    Minion did not return. [No response]
[DEBUG   ] return event: {u'<es2_ip>': {u'failed': True}}
[DEBUG   ] LazyLoaded localfs.init_kwargs
[DEBUG   ] LazyLoaded no_return.output
<es2_ip>:
    Minion did not return. [No response]
[DEBUG   ] return event: {u'<es1_ip>': {u'failed': True}}
[DEBUG   ] LazyLoaded localfs.init_kwargs
[DEBUG   ] LazyLoaded no_return.output
<es1_ip>:
    Minion did not return. [No response]
[DEBUG   ] return event: {u'<es2_ip>': {u'failed': True}}
[DEBUG   ] LazyLoaded localfs.init_kwargs
[DEBUG   ] LazyLoaded no_return.output
<es2_ip>:
    Minion did not return. [No response]
```

### 缓解
问了下反馈问题的同学，问题什么时候发现的，同学说到昨天刚发现的，这边想到本周一刚升过salt的版本，说不定是minion的兼容性问题引起的。

于是对`<es1_ip>`这台服务器的salt-minion进行升级，升完后重启进程，再对分组test.ping，果然升完版本的minion正常返回了，但老版本的依旧未响应。
```
# salt -N ELK test.ping
[DEBUG   ] Configuration file path: /etc/salt/master
[WARNING ] Insecure logging configuration detected! Sensitive data may be logged.
[DEBUG   ] Reading configuration from /etc/salt/master
[DEBUG   ] Including configuration from '/etc/salt/master.d/api.conf'
[DEBUG   ] Reading configuration from /etc/salt/master.d/api.conf
[DEBUG   ] Including configuration from '/etc/salt/master.d/eauth.conf'
[DEBUG   ] Reading configuration from /etc/salt/master.d/eauth.conf
[DEBUG   ] Including configuration from '/etc/salt/master.d/nodegroups.conf'
[DEBUG   ] Reading configuration from /etc/salt/master.d/nodegroups.conf
[DEBUG   ] Using cached minion ID from /etc/salt/minion_id: <salt-master_ip>
[DEBUG   ] Missing configuration file: /root/.saltrc
[DEBUG   ] MasterEvent PUB socket URI: /var/run/salt/master/master_event_pub.ipc
[DEBUG   ] MasterEvent PULL socket URI: /var/run/salt/master/master_event_pull.ipc
[DEBUG   ] nodegroup_comp(ELK) => [u'(', u'L@<es1_ip>,<es2_ip>,', u')']
[DEBUG   ] No nested nodegroups detected. Using original nodegroup definition: L@<es1_ip>,<es2_ip>,
[DEBUG   ] Initializing new AsyncZeroMQReqChannel for (u'/etc/salt/pki/master', u'<salt-master_ip>_master', u'tcp://127.0.0.1:4506', u'clear')
[DEBUG   ] Connecting the Minion to the Master URI (for the return server): tcp://127.0.0.1:4506
[DEBUG   ] Trying to connect to: tcp://127.0.0.1:4506
[DEBUG   ] Initializing new IPCClient for path: /var/run/salt/master/master_event_pub.ipc
[DEBUG   ] LazyLoaded local_cache.get_load
[DEBUG   ] Reading minion list from /var/cache/salt/master/jobs/6a/b5dc9bfa3189f2e5a2801fa11c7795347291571f22738fb634187aa8348ba1/.minions.p
[DEBUG   ] get_iter_returns for jid 20180719161320645348 sent to set([]) will timeout at 16:15:20.668820
[DEBUG   ] jid 20180719161320645348 return from <es1_ip>
[DEBUG   ] return event: {u'<es1_ip>': {u'jid': u'20180719161320645348', u'retcode': 0, u'ret': True}}
[DEBUG   ] LazyLoaded nested.output
<es1_ip>:
    True
[DEBUG   ] Checking whether jid 20180719161320645348 is still running
[DEBUG   ] Initializing new AsyncZeroMQReqChannel for (u'/etc/salt/pki/master', u'<salt-master_ip>_master', u'tcp://127.0.0.1:4506', u'clear')
[DEBUG   ] Connecting the Minion to the Master URI (for the return server): tcp://127.0.0.1:4506
[DEBUG   ] Trying to connect to: tcp://127.0.0.1:4506
[DEBUG   ] Passing on saltutil error. Key 'u'retcode' missing from client return. This may be an error in the client.
[DEBUG   ] Passing on saltutil error. Key 'u'retcode' missing from client return. This may be an error in the client.
[DEBUG   ] return event: {u'<es2_ip>': {u'failed': True}}
[DEBUG   ] LazyLoaded localfs.init_kwargs
[DEBUG   ] LazyLoaded localfs.init_kwargs
[DEBUG   ] LazyLoaded no_return.output
<es2_ip>:
    Minion did not return. [No response]
[DEBUG   ] return event: {u'<es2_ip>': {u'failed': True}}
[DEBUG   ] LazyLoaded localfs.init_kwargs
[DEBUG   ] LazyLoaded no_return.output
<es2_ip>:
    Minion did not return. [No response]
```

到这边就可以确定了，确实是因为salt-minion版本不兼容所导致。
- salt-master: salt-master-2018.3.2-1.el6
- salt-minion: salt-minion-2015.5.11-1.el6

从DEBUG日志中可以看到关键信息：

> No nested nodegroups detected. Using original nodegroup definition

根据这句google下，发现也有人提过类似问题的issue：#[39270][1]，官方以老版本不在提供支持为由关闭了问题。

### 后记
salt-master版本升级这么大的动作，说完全没有影响也是不可能的，当时在变更的时候就担心过salt-minion不升的兼容性问题，好在核心功能并不影响。

想懒不升minion看来还是不行，下周该要计划产线上将近2K台机器的salt-minion升级工作了。

[1]: https://github.com/saltstack/salt/issues/39270