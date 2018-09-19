---
layout: post
title: "bash warning setlocale LC_ALL cannot change locale问题解决记录"
subtitle: "locale/yum/rpm填坑记录"
date: 2018-09-19 10:17AM
catalog: true
tags:
    - 运维
    - Linux
---

### 背景
同事反馈有台服务器中文显示有问题，搞了几天都没能解决，反馈到了我这边。

登陆上服务器后，具体现象如下：

- 登录服务器后，console提示语言分配异常

```
$ ssh X.X.X.X
Last login: Tue Sep 18 21:09:55 2018 from 192.168.13.14
Authorized uses only. All activity may be 	monitored and reported.
-bash: warning: setlocale: LC_ALL: cannot change locale (en_US.utf-8)
-bash: warning: setlocale: LC_ALL: cannot change locale (en_US.utf-8)
/bin/sh: warning: setlocale: LC_ALL: cannot change locale (en_US.utf-8)
```

- 查看用户上传的带有中文的文件夹，无法正常显示

```
# ls /path/to/folder/
ADMIN-New(admin??????)(admin.cctalk.com)
ADMIN-New(admin??????)(admin.cctalk.com)@tmp
JAVA-Job(?????????????????????)(ccjob.soa.yeshj.com)
JAVA-Job(?????????????????????)(ccjob.soa.yeshj.com)@tmp
```

### 问题

先查系统版本：

```
#cat /etc/redhat-release 
CentOS Linux release 7.1.1503 (Core) 
```

网上搜了下，一般来说，修改Linux系统locale配置就能恢复正常了，但我们这次的情况比较复杂，无论怎么改locale配置，均告失败，下面记录下调整过locale配置的文件：

- ~/.zshrc
- ~/.bashrc
- /etc/profile
- /etc/locale.conf
- /etc/sysconfig/i18n

无论改哪个，发现效果都一样，还是提示`cannot change locale`，这是怎么回事呢？

查看系统所有支持的`locale`库，终于发现了问题：

```
$locale -a
C
POSIX
```

这是什么情况？所有国家地区的语言库都缺失，只剩下操作系统自带的`POSIX`语言库了，也就是说，语言库丢了，自然是显示不出英语之外任何字体了！

尝试重新安装locale语言库，但还是失败：

```
#localedef -v -c -i en_US -f UTF-8 en_US.UTF-8
/usr/share/i18n/locales/en_US:7: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:8: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:9: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:11: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:14: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:15: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:16: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:17: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:19: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:20: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:21: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:22: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:23: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:24: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:25: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:26: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:27: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:28: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:29: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:34: non-symbolic character value should not be used
/usr/share/i18n/locales/en_GB:50: non-symbolic character value should not be used
/usr/share/i18n/locales/i18n:1425: non-symbolic character value should not be used
/usr/share/i18n/locales/i18n:1674: non-symbolic character value should not be used
/usr/share/i18n/locales/i18n:1719: non-symbolic character value should not be used
/usr/share/i18n/locales/i18n:1756: non-symbolic character value should not be used
/usr/share/i18n/locales/en_GB:53: non-symbolic character value should not be used
/usr/share/i18n/locales/en_GB:59: non-symbolic character value should not be used
/usr/share/i18n/locales/en_GB:152: non-symbolic character value should not be used
/usr/share/i18n/locales/en_US:40: non-symbolic character value should not be used
/usr/share/i18n/locales/iso14651_t1:3: non-symbolic character value should not be used
/usr/share/i18n/locales/translit_neutral:10: non-symbolic character value should not be used
/usr/share/i18n/locales/translit_neutral:11: non-symbolic character value should not be used
/usr/share/i18n/locales/translit_neutral:12: non-symbolic character value should not be used
/usr/share/i18n/locales/translit_neutral:13: non-symbolic character value should not be used
/usr/share/i18n/locales/translit_neutral:14: non-symbolic character value should not be used
/usr/share/i18n/locales/translit_neutral:15: non-symbolic character value should not be used
/usr/share/i18n/locales/translit_neutral:16: non-symbolic character value should not be used
/usr/share/i18n/locales/translit_neutral:17: non-symbolic character value should not be used
LC_NAME: field `name_gen' not defined
LC_IDENTIFICATION: field `audience' not defined
LC_IDENTIFICATION: field `application' not defined
LC_IDENTIFICATION: field `abbreviation' not defined
LC_IDENTIFICATION: no identification for category `LC_MEASUREMENT'
LC_CTYPE: table for class "upper": 1756 bytes
LC_CTYPE: table for class "lower": 1756 bytes
LC_CTYPE: table for class "alpha": 4320 bytes
LC_CTYPE: table for class "digit": 600 bytes
LC_CTYPE: table for class "xdigit": 600 bytes
LC_CTYPE: table for class "space": 856 bytes
LC_CTYPE: table for class "print": 5976 bytes
LC_CTYPE: table for class "graph": 5976 bytes
LC_CTYPE: table for class "blank": 856 bytes
LC_CTYPE: table for class "cntrl": 664 bytes
LC_CTYPE: table for class "punct": 4824 bytes
LC_CTYPE: table for class "alnum": 4320 bytes
LC_CTYPE: table for class "combining": 3152 bytes
LC_CTYPE: table for class "combining_level3": 2832 bytes
LC_CTYPE: table for map "toupper": 16924 bytes
LC_CTYPE: table for map "tolower": 15388 bytes
LC_CTYPE: table for map "totitle": 16924 bytes
LC_CTYPE: table for width: 26712 bytes
cannot create temporary file: No such file or directory
```

网上找到[一篇文章][1]提到与操作系统的`glibc`有关，单独查看了下`glibc`的[官方包][2]，发现确实有和语言库相关的内容：

```
/usr/share/i18n/locales/zh_CN
/usr/share/locale/zh_CN/
/usr/share/locale/zh_CN/LC_MESSAGES/libc.mo
```

终于找到问题原因，后面就是如何安装/升级`glibc`的工作了。

### 缓解

#### yum安装glibc

想要安装/升级`glibc`，第一反应是直接yum安装，但这时候碰到了两件吐血的事情：

##### python升级导致yum无法使用

直接调用`yum`命令，如下报错：

```
#yum --help
There was a problem importing one of the Python modules
required to run yum. The error leading to this problem was:

   No module named yum

Please install a package which provides this module, or
verify that the module is installed correctly.

It's possible that the above module doesn't match the
current version of Python, which is:
2.7.13 (default, Jan  9 2017, 22:30:55) 
[GCC 4.8.3 20140911 (Red Hat 4.8.3-9)]

If you cannot solve this problem yourself, please go to 
the yum faq at:
  http://yum.baseurl.org/wiki/Faq
  
```

原因是`yum`命令调用的`python`环境被动过了，导致无法正常使用`yum`，发现这台服务器上有4个`python`，找系统自带的`python`也是找了大半天：

```
#/usr/bin/python2 -V
Python 2.7.5

#/opt/ActivePython-2.7/bin/python -V
Python 2.7.12

#/usr/local/python2/bin/python -V
Python 2.7.13

#/usr/bin/python -V
Python 2.7.13

#/usr/local/python3/bin/python3 -V
Python 3.5.2

#/usr/bin/python2
Python 2.7.5 (default, Jun 17 2014, 18:11:42) 
[GCC 4.8.2 20140120 (Red Hat 4.8.2-16)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import yum
>>> 
```

##### 无源可yum

由于CentOS系统的生命周期，官方已不再提供7.1操作系统的支持，对应的仓库也是被移除了：

![img](/img/in-post/post-180919-linux-locale-issue/WechatIMG2123.png)

强配默认CentOS7的源只会去仓库取7.5系统的包，当然是不行的，提示`404`错误，找不到7.1需要的包：

![img](/img/in-post/post-180919-linux-locale-issue/WechatIMG2125.jpeg)

但好在`yum`把所有升级`glibc`的包都显示出来了，我们可以从[CentOS 7.1历史仓库][3]中找到这些对应的rpm包：

- glibc.x86_64
- glibc-common.x86_64
- glibc-devel
- glibc-headers

#### rpm安装glibc

##### rpm不可用

很狗血的是，`rpm`也不可用，调`rpm`命令抛错：

```
#rpm -V rpm
error: Unable to open /usr/lib/rpm/rpmrc for reading: No such file or directory.
```

`yum`不可用，`rpm`不可用，鸡和蛋都没有，尴尬！

实在没办法了，碰下运气，线上找一台系统版本一样的服务器，把rpm包所有文件打包拷贝过来，最后居然成功了：

```
#cd /usr/lib/
#wget ftp://xxx/rpm.tar.gz
#tar zxf rpm.tar.gz
#cd rpm/
#rpm -V rpm
missing     /usr/lib/tmpfiles.d/rpm.conf
#mkdir -p /usr/lib/tmpfiles.d/
#cd /usr/lib/tmpfiles.d/
#wget ftp://xxx/rpm.conf
#rpm -V rpm
.......T.    /usr/lib/tmpfiles.d/rpm.conf
```

##### 升级glibc

修复好`rpm`命令后，我们开始升级`glibc`库：

```
#rpm -qa|grep glibc
glibc-common-2.17-78.el7.x86_64
glibc-devel-2.17-78.el7.x86_64
glibc-2.17-78.el7.x86_64
glibc-headers-2.17-78.el7.x86_64

# cd /usr/local/src/
# wget http://mirrors.cqu.edu.cn/CentOS/7.5.1804/os/x86_64/Packages/glibc-2.17-222.el7.x86_64.rpm
# wget http://mirrors.cqu.edu.cn/CentOS/7.5.1804/os/x86_64/Packages/glibc-common-2.17-222.el7.x86_64.rpm
# wget http://mirrors.cqu.edu.cn/CentOS/7.5.1804/os/x86_64/Packages/glibc-devel-2.17-222.el7.x86_64.rpm
# wget http://mirrors.cqu.edu.cn/CentOS/7.5.1804/os/x86_64/Packages/glibc-headers-2.17-222.el7.x86_64.rpm

#rpm -Uvh --replacepkgs glibc*.rpm
Preparing...                          ################################# [100%]
Updating / installing...
   1:glibc-common-2.17-222.el7        ################################# [ 13%]
   2:glibc-2.17-222.el7               warning: /etc/nsswitch.conf created as /etc/nsswitch.conf.rpmnew
################################# [ 25%]
   3:glibc-headers-2.17-222.el7       ################################# [ 38%]
   4:glibc-devel-2.17-222.el7         ################################# [ 50%]
Cleaning up / removing...
   5:glibc-devel-2.17-78.el7          ################################# [ 63%]
   6:glibc-headers-2.17-78.el7        ################################# [ 75%]
   7:glibc-common-2.17-78.el7         ################################# [ 88%]
   8:glibc-2.17-78.el7                ################################# [100%]

#rpm -qa|grep glibc
glibc-2.17-222.el7.x86_64
glibc-common-2.17-222.el7.x86_64
glibc-headers-2.17-222.el7.x86_64
glibc-devel-2.17-222.el7.x86_64
```

适当调整`locale`文件后，再次查看目录，文件夹名已经能够正常显示了：

```
# cat /etc/locale.conf
export LC_ALL=zh_CN.UTF-8
export LANG=zh_CN.UTF-8

# tail /etc/profile -n1
export LC_ALL=zh_CN.UTF-8

#cat /etc/sysconfig/i18n
LC_CTYPE="zh_CN.UTF-8"

# ls /path/to/folder/
ADMIN-New(admin后台)(admin.cctalk.com)
ADMIN-New(admin后台)(admin.cctalk.com)@tmp
JAVA-Job(任务执行微服务)(ccjob.soa.yeshj.com)
JAVA-Job(任务执行微服务)(ccjob.soa.yeshj.com)@tmp
```

### 后记

具体为啥这台机器有这么多Python环境，`yum`和`rpm`为啥都不可用，但好在还能救回来。

无论是产线还是测试环境服务器，权限一定要控制起来，除非确认即便是搞坏了重装也无所谓，否则搞坏了耽误的就是大家的时间了。

[1]: https://www.linuxquestions.org/questions/linux-general-1/locale-cannot-set-lc_all-to-default-locale-no-such-file-or-directory-218622/ "locale: Cannot Set LC_ALL to default locale: No such file or directory."
[2]: https://archlinux.pkgs.org/rolling/archlinux-core-x86_64/glibc-2.28-4-x86_64.pkg.tar.xz.html "glibc-2.28-4-x86_64.pkg.tar.xz"
[3]: http://vault.centos.org/7.1.1503/os/x86_64/Packages/