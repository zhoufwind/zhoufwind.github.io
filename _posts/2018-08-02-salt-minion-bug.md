---
layout: post
title: "记salt-minion升级后发现的两个BUG"
subtitle: "salt-minion SALT.STATES.ARCHIVE模块调整及重启salt-minion会kill其他进程问题"
date: 2018-08-02 05:02PM
catalog: true
tags:
    - 运维
    - Salt
---
### 背景
升级背景在前文中也有提到，salt-master升级，salt-minion跟着一起升，非必要。

但在本次升级过程中发现的两个问题，发现升级salt-minion又变得是十分必要的了，具体原因详见以下问题：

### 问题

#### SALT.STATES.ARCHIVE模块改写，不向下兼容

接用户反馈，新装机交付的服务器搜不到zabbix监控。问题分别被提交到监控同学、装机同学、自动化装机系统开发同学，最后到我这边，反馈装机时调用部署zabbix的SLS文件异常。

获取到有问题的服务器IP后，手工执行尝试重现问题，可以重现，并看到报错信息：

```
        "archive_|-Update_zabbix_bin_|-/zabbix/install/path/zabbix/bin_|-extracted": {
            "changes": {},
            "start_time": "15:13:56.969267",
            "name": "/zabbix/install/path/zabbix/bin",
            "__run_num__": 2,
            "__sls__": "sls_lin_update_zabbix",
            "__id__": "Update_zabbix_bin",
            "result": False,
            "comment": "Archive does not have a single top-level directory. To allow this archive to be extracted, set "enforce_toplevel" to False. To avoid a "tar-bomb" it may also be advisable to set a top-level directory by adding it to the "name" value (for example, setting "name" to /zabbix/install/path/zabbix/bin/some_dir instead of /zabbix/install/path/zabbix/bin/).",
            "duration": 24.411
        },
```
具体问题应该是在：`comment`中返回的异常，谷歌搜了下关键字：`Archive does not have a single top-level directory`，发现有人遇到类似问题，详见#[34101][1]，然后在2016.11.0后版本均做更新，可以再官方文档[SALT.STATES.ARCHIVE][2]模块上找到对应报错说明：  
![img](/img/in-post/post-salt-minion-bug/SALT.STATES.ARCHIVE.png)

看文档说明是为了防止解压造成TAR文件炸弹而设置，如果想要正常使用解压功能，必须单独配置`enforce_toplevel`为`False`，调整后SLS恢复正常。

#### CentOS7 systemd误杀由salt起动的服务

接用户反馈，在重启salt-minion服务后，有业务服务进程突然消失的情况，影响到线上。

产线服务器上测了一把，可以重现，并且确定误杀的进程都是在`salt-minion.service`这个主进程下的。

如下，`python /home/scripts/server_load_ctrl.py start`该进程是通过salt启动的，重启salt-minion服务后，该python进程也消失了：

```
# systemctl status salt-minion.service 
● salt-minion.service - The Salt Minion
   Loaded: loaded (/usr/lib/systemd/system/salt-minion.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2018-07-17 15:26:39 CST; 2 weeks 2 days ago
 Main PID: 1287 (salt-minion)
    Tasks: 8
   Memory: 46.8M
   CGroup: /system.slice/salt-minion.service
           ├─1287 /usr/bin/python /usr/bin/salt-minion
           ├─1490 /usr/bin/python /usr/bin/salt-minion
           └─7094 python /home/scripts/server_load_ctrl.py start
# systemctl restart salt-minion.service 
# systemctl status salt-minion.service 
● salt-minion.service - The Salt Minion
   Loaded: loaded (/usr/lib/systemd/system/salt-minion.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2018-08-03 10:54:58 CST; 7s ago
 Main PID: 28902 (salt-minion)
    Tasks: 7
   Memory: 36.6M
   CGroup: /system.slice/salt-minion.service
           ├─28902 /usr/bin/python /usr/bin/salt-minion
           └─28906 /usr/bin/python /usr/bin/salt-minion
```

可以发现，原pid为7094的python进程一同被kill掉了。重启salt-minion后，salt-minion有，但python进程没了。

谷歌上搜搜了下关键字`salt LimitNOFILE`，发现已经有人遇到类似问题了：《重启salt-minion会连同kill其他进程》[链接戳我][3]，其中引用了官方的issue#[22993][4]。

以下为原salt-minion启动脚本：

```
# cat /usr/lib/systemd/system/salt-minion.service
[Unit]
Description=The Salt Minion
After=syslog.target network.target

[Service]
Type=simple
ExecStart=/usr/bin/salt-minion

[Install]
WantedBy=multi-user.target
```

新版已更新了salt-minion启动脚本，具体内容如下：

```
# cat /usr/lib/systemd/system/salt-minion.service
[Unit]
Description=The Salt Minion
Documentation=man:salt-minion(1) file:///usr/share/doc/salt/html/contents.html https://docs.saltstack.com/en/latest/contents.html
After=network.target salt-master.service

[Service]
KillMode=process
Type=notify
NotifyAccess=all
LimitNOFILE=8192
ExecStart=/usr/bin/salt-minion

[Install]
WantedBy=multi-user.target
```

我们发现，`Service`这项在原来的文件上多了两个参数：`KillMode=process`及`LimitNOFILE=8192`，这两项都保证了只停止主进程。

网上搜了下`KillMode`相关文章，以下这篇文章《Systemd 入门教程：实战篇》[链接戳我][5]，做了比较详细的解释：

![img](/img/in-post/post-salt-minion-bug/systemd-killmode.png)

更新salt-minion版本后，发现问题确实已修复，重启salt-minion后，python及java服务都正常运行：

![img](/img/in-post/post-salt-minion-bug/systemctl-restart-1.png)

![img](/img/in-post/post-salt-minion-bug/systemctl-restart-2.png)

### 后记
原本以为升完salt-master，salt-minion升不升也无所谓，但时间越久，发现因版本兼容性导致的问题也越多，所以线上salt服务端及客户端尽量版本要保持一致，减少幺蛾子。

[1]: https://github.com/saltstack/salt/issues/34101
[2]: https://docs.saltstack.com/en/latest/ref/states/all/salt.states.archive.html
[3]: https://xieyugui.wordpress.com/2017/09/13/%E9%87%8D%E5%90%AFsalt-minion%E4%BC%9A%E8%BF%9E%E5%90%8Ckill%E5%85%B6%E4%BB%96%E8%BF%9B%E7%A8%8B/
[4]: https://github.com/saltstack/salt/issues/22993
[5]: http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-part-two.html