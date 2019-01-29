---
layout: post
title: "pycurl错误导致yum失败"
subtitle: "yum失败问题处理记录"
date: 2019-01-29 01:51PM
catalog: true
tags:
    - 运维
    - Linux
---

### 背景

测试环境某台服务器salt不通，尝试登录服务器手工启动salt-minion，后发现salt-minion程序未安装，于是开始安装salt-minion，但发现安装过程中出现异常。

### 问题 & 缓解

#### urlgrabber

首次yum安装，提示`ImportError: No module named urlgrabber.grabber`，这步比较好解决，pip直接安装模块即可：

```
# yum install salt-minion
Traceback (most recent call last):
  File "/usr/libexec/urlgrabber-ext-down", line 22, in <module>
    from urlgrabber.grabber import \
ImportError: No module named urlgrabber.grabber
Traceback (most recent call last):
  File "/usr/libexec/urlgrabber-ext-down", line 22, in <module>
    from urlgrabber.grabber import \
ImportError: No module named urlgrabber.grabber
Traceback (most recent call last):
  File "/usr/libexec/urlgrabber-ext-down", line 22, in <module>
    from urlgrabber.grabber import \
ImportError: No module named urlgrabber.grabber
Traceback (most recent call last):
  File "/usr/libexec/urlgrabber-ext-down", line 22, in <module>
    from urlgrabber.grabber import \
ImportError: No module named urlgrabber.grabber
Traceback (most recent call last):
  File "/usr/libexec/urlgrabber-ext-down", line 22, in <module>
    from urlgrabber.grabber import \
ImportError: No module named urlgrabber.grabber

# pip install urlgrabber
Collecting urlgrabber
  Downloading https://files.pythonhosted.org/packages/29/1a/f509987826e17369c52a80a07b257cc0de3d7864a303175f2634c8bcb3e3/urlgrabber-3.10.2.tar.gz (84kB)
    100% |████████████████████████████████| 92kB 55kB/s 
Building wheels for collected packages: urlgrabber
  Running setup.py bdist_wheel for urlgrabber ... done
  Stored in directory: /root/.cache/pip/wheels/09/3c/6b/e1326d285aa3f111a73829217a10be6dc4ea6c23aeb2794242
Successfully built urlgrabber
Installing collected packages: urlgrabber
Successfully installed urlgrabber-3.10.2
```

#### pycurl

第二次安装提示`ImportError: No module named pycurl`，主要问题在于这个模块但安装，直接pip安装模块失败，提示：`__main__.ConfigurationError: Could not run curl-config: [Errno 2] No such file or directory`：

```
# yum install salt-minion
Traceback (most recent call last):
  File "/usr/libexec/urlgrabber-ext-down", line 22, in <module>
    from urlgrabber.grabber import \
  File "/usr/local/python2.7.12/lib/python2.7/site-packages/urlgrabber/__init__.py", line 55, in <module>
    from grabber import urlgrab, urlopen, urlread
  File "/usr/local/python2.7.12/lib/python2.7/site-packages/urlgrabber/grabber.py", line 532, in <module>
    import pycurl
ImportError: No module named pycurl
Traceback (most recent call last):
  File "/usr/libexec/urlgrabber-ext-down", line 22, in <module>
    from urlgrabber.grabber import \
  File "/usr/local/python2.7.12/lib/python2.7/site-packages/urlgrabber/__init__.py", line 55, in <module>
    from grabber import urlgrab, urlopen, urlread
  File "/usr/local/python2.7.12/lib/python2.7/site-packages/urlgrabber/grabber.py", line 532, in <module>
    import pycurl
ImportError: No module named pycurl
Traceback (most recent call last):
  File "/usr/libexec/urlgrabber-ext-down", line 22, in <module>
    from urlgrabber.grabber import \
  File "/usr/local/python2.7.12/lib/python2.7/site-packages/urlgrabber/__init__.py", line 55, in <module>
    from grabber import urlgrab, urlopen, urlread
  File "/usr/local/python2.7.12/lib/python2.7/site-packages/urlgrabber/grabber.py", line 532, in <module>
    import pycurl
ImportError: No module named pycurl
Traceback (most recent call last):
  File "/usr/libexec/urlgrabber-ext-down", line 22, in <module>
    from urlgrabber.grabber import \
  File "/usr/local/python2.7.12/lib/python2.7/site-packages/urlgrabber/__init__.py", line 55, in <module>
    from grabber import urlgrab, urlopen, urlread
  File "/usr/local/python2.7.12/lib/python2.7/site-packages/urlgrabber/grabber.py", line 532, in <module>
    import pycurl
ImportError: No module named pycurl
Traceback (most recent call last):
  File "/usr/libexec/urlgrabber-ext-down", line 22, in <module>
    from urlgrabber.grabber import \
  File "/usr/local/python2.7.12/lib/python2.7/site-packages/urlgrabber/__init__.py", line 55, in <module>
    from grabber import urlgrab, urlopen, urlread
  File "/usr/local/python2.7.12/lib/python2.7/site-packages/urlgrabber/grabber.py", line 532, in <module>
    import pycurl
ImportError: No module named pycurl

# pip install pycurl
Collecting pycurl
  Downloading https://files.pythonhosted.org/packages/e8/e4/0dbb8735407189f00b33d84122b9be52c790c7c3b25286826f4e1bdb7bde/pycurl-7.43.0.2.tar.gz (214kB)
    100% |████████████████████████████████| 215kB 179kB/s 
    Complete output from command python setup.py egg_info:
    Traceback (most recent call last):
      File "<string>", line 1, in <module>
      File "/tmp/pip-install-WRX9VZ/pycurl/setup.py", line 913, in <module>
        ext = get_extension(sys.argv, split_extension_source=split_extension_source)
      File "/tmp/pip-install-WRX9VZ/pycurl/setup.py", line 582, in get_extension
        ext_config = ExtensionConfiguration(argv)
      File "/tmp/pip-install-WRX9VZ/pycurl/setup.py", line 99, in __init__
        self.configure()
      File "/tmp/pip-install-WRX9VZ/pycurl/setup.py", line 227, in configure_unix
        raise ConfigurationError(msg)
    __main__.ConfigurationError: Could not run curl-config: [Errno 2] No such file or directory
    
    ----------------------------------------
Command "python setup.py egg_info" failed with error code 1 in /tmp/pip-install-WRX9VZ/pycurl/
```

发现`curl-config`该命令确实不存在，但`curl`可用：

```
# curl-config
-bash: curl-config: command not found

# curl -V
curl 7.29.0 (x86_64-redhat-linux-gnu) libcurl/7.29.0 NSS/3.21 Basic ECC zlib/1.2.7 libidn/1.28 libssh2/1.4.3
Protocols: dict file ftp ftps gopher http https imap imaps ldap ldaps pop3 pop3s rtsp scp sftp smtp smtps telnet tftp 
Features: AsynchDNS GSS-Negotiate IDN IPv6 Largefile NTLM NTLM_WB SSL libz unix-sockets 
```

[网上][1]搜索后发现，有文章指出需要先编译安装[`curl`][2]：

```
# which curl
/bin/curl
# cp /bin/curl /bin/curl.bak
# cd /usr/local/src/
# wget https://curl.haxx.se/download/curl-7.63.0.tar.gz
# tar zxf curl-7.63.0.tar.gz
# cd curl-7.63.0/
# ./configure
# make
# make install
Making install in lib
make[1]: Entering directory `/usr/local/src/curl-7.63.0/lib'
make[2]: Entering directory `/usr/local/src/curl-7.63.0/lib'
 /bin/mkdir -p '/usr/local/lib'
 /bin/sh ../libtool   --mode=install /bin/install -c   libcurl.la '/usr/local/lib'
libtool: install: /bin/install -c .libs/libcurl.so.4.5.0 /usr/local/lib/libcurl.so.4.5.0
libtool: install: (cd /usr/local/lib && { ln -s -f libcurl.so.4.5.0 libcurl.so.4 || { rm -f libcurl.so.4 && ln -s libcurl.so.4.5.0 libcurl.so.4; }; })
libtool: install: (cd /usr/local/lib && { ln -s -f libcurl.so.4.5.0 libcurl.so || { rm -f libcurl.so && ln -s libcurl.so.4.5.0 libcurl.so; }; })
libtool: install: /bin/install -c .libs/libcurl.lai /usr/local/lib/libcurl.la
libtool: install: /bin/install -c .libs/libcurl.a /usr/local/lib/libcurl.a
libtool: install: chmod 644 /usr/local/lib/libcurl.a
libtool: install: ranlib /usr/local/lib/libcurl.a
libtool: finish: PATH="/opt/kube/bin:/opt/kube/bin:/opt/kube/bin:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin:/sbin" ldconfig -n /usr/local/lib
----------------------------------------------------------------------
Libraries have been installed in:
   /usr/local/lib
...

# curl-config --version
libcurl 7.63.0

# curl -V
curl 7.29.0 (x86_64-redhat-linux-gnu) libcurl/7.63.0 OpenSSL/1.0.2k zlib/1.2.7
Protocols: dict file ftp ftps gopher http https imap imaps pop3 pop3s rtsp smb smbs smtp smtps telnet tftp 
Features: AsynchDNS IPv6 Largefile NTLM NTLM_WB SSL libz unix-sockets 
```

`curl-config`安装完成后，再编译或pip安装[`pycurl`][3]：

```
# cd /usr/local/src/
# wget https://files.pythonhosted.org/packages/e8/e4/0dbb8735407189f00b33d84122b9be52c790c7c3b25286826f4e1bdb7bde/pycurl-7.43.0.2.tar.gz
# tar zxf pycurl-7.43.0.2.tar.gz
# cd pycurl-7.43.0.2/
# python setup.py install --curl-config=/usr/local/bin/curl-config
...
Installed /usr/local/python2.7.12/lib/python2.7/site-packages/pycurl-7.43.0.2-py2.7-linux-x86_64.egg
Processing dependencies for pycurl==7.43.0.2
Finished processing dependencies for pycurl==7.43.0.2
```

安装完成后yum安装仍报错，提示当前libcurl版本（7.29.0）低于刚编译的libcurl版本（7.63.0）：

```
# yum install salt-minion
Traceback (most recent call last):
  File "/usr/libexec/urlgrabber-ext-down", line 22, in <module>
    from urlgrabber.grabber import \
  File "/usr/local/python2.7.12/lib/python2.7/site-packages/urlgrabber/__init__.py", line 55, in <module>
    from grabber import urlgrab, urlopen, urlread
  File "/usr/local/python2.7.12/lib/python2.7/site-packages/urlgrabber/grabber.py", line 532, in <module>
    import pycurl
  File "build/bdist.linux-x86_64/egg/pycurl.py", line 7, in <module>
  File "build/bdist.linux-x86_64/egg/pycurl.py", line 6, in __bootstrap__
ImportError: pycurl: libcurl link-time version (7.29.0) is older than compile-time version (7.63.0)
Traceback (most recent call last):
  File "/usr/libexec/urlgrabber-ext-down", line 22, in <module>
    from urlgrabber.grabber import \
  File "/usr/local/python2.7.12/lib/python2.7/site-packages/urlgrabber/__init__.py", line 55, in <module>
    from grabber import urlgrab, urlopen, urlread
  File "/usr/local/python2.7.12/lib/python2.7/site-packages/urlgrabber/grabber.py", line 532, in <module>
    import pycurl
  File "build/bdist.linux-x86_64/egg/pycurl.py", line 7, in <module>
  File "build/bdist.linux-x86_64/egg/pycurl.py", line 6, in __bootstrap__
ImportError: pycurl: libcurl link-time version (7.29.0) is older than compile-time version (7.63.0)
```

[网上][4]搜索了下，发现有人也遇到过类似问题，解决方法为手工拷贝`libcurl`文件至系统库目录：

```
# mv /usr/lib64/libcurl.so.4 /usr/lib64/libcurl.so.4.bak
# ln -s /usr/local/lib/libcurl.so.4.5.0 /usr/lib64/libcurl.so.4
# ll /usr/lib64/libcurl.so.4
lrwxrwxrwx 1 root root 31 Jan 29 13:44 /usr/lib64/libcurl.so.4 -> /usr/local/lib/libcurl.so.4.5.0
```

建立软链后yum安装恢复正常：

```
# yum install salt-minion
...
Installed:
  salt-minion.noarch 0:2018.3.3-1.el7                                                                                                                                                                     

Dependency Installed:
  python-psutil.x86_64 0:2.2.1-1.el7                                python-tornado.x86_64 0:4.2.1-4.el7                                python2-futures.noarch 0:3.1.1-5.el7                               

Dependency Updated:
  salt.noarch 0:2018.3.3-1.el7                                                                                                                                                                            

Complete!
```

#### salt-minion

以默认的python启动会报缺少salt模块的错误，需要制定python全路径后才能正常启动：

```
# systemctl start salt-minion.service 
Job for salt-minion.service failed because the control process exited with error code. See "systemctl status salt-minion.service" and "journalctl -xe" for details.

# systemctl status salt-minion.service
● salt-minion.service - The Salt Minion
   Loaded: loaded (/usr/lib/systemd/system/salt-minion.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Tue 2019-01-29 14:34:37 CST; 6s ago
     Docs: man:salt-minion(1)
           file:///usr/share/doc/salt/html/contents.html
           https://docs.saltstack.com/en/latest/contents.html
  Process: 164010 ExecStart=/usr/bin/salt-minion (code=exited, status=1/FAILURE)
 Main PID: 164010 (code=exited, status=1/FAILURE)

Jan 29 14:34:37 pr-soa-gateway-35-70.hjidc.com systemd[1]: Starting The Salt Minion...
Jan 29 14:34:37 pr-soa-gateway-35-70.hjidc.com salt-minion[164010]: Traceback (most recent call last):
Jan 29 14:34:37 pr-soa-gateway-35-70.hjidc.com salt-minion[164010]: File "/usr/bin/salt-minion", line 6, in <module>
Jan 29 14:34:37 pr-soa-gateway-35-70.hjidc.com salt-minion[164010]: import salt.utils.platform
Jan 29 14:34:37 pr-soa-gateway-35-70.hjidc.com salt-minion[164010]: ImportError: No module named salt.utils.platform
Jan 29 14:34:37 pr-soa-gateway-35-70.hjidc.com systemd[1]: salt-minion.service: main process exited, code=exited, status=1/FAILURE
Jan 29 14:34:37 pr-soa-gateway-35-70.hjidc.com systemd[1]: Failed to start The Salt Minion.
Jan 29 14:34:37 pr-soa-gateway-35-70.hjidc.com systemd[1]: Unit salt-minion.service entered failed state.
Jan 29 14:34:37 pr-soa-gateway-35-70.hjidc.com systemd[1]: salt-minion.service failed.

# /usr/bin/python /usr/bin/salt-minion -d
Traceback (most recent call last):
  File "/usr/bin/salt-minion", line 6, in <module>
    import salt.utils.platform
ImportError: No module named salt.utils.platform

# python2.7 /usr/bin/salt-minion -d

# ps -ef|grep salt
root     164187      1 20 14:35 ?        00:00:00 python2.7 /usr/bin/salt-minion -d
root     164375 119433  0 14:35 pts/0    00:00:00 grep --color=auto salt
```

### 后记

这个问题还是由于开发人员私自升级python，破坏了线上标准环境，如需升级python服务，需联系运维人员，统一安装虚拟python环境，以避免对系统环境的破坏。

[1]: https://www.douban.com/note/205693536/ "python安装pycurl错误 Exception: `curl-config' not found -- please install the libcurl development files"
[2]: https://curl.haxx.se/download.html
[3]: https://pypi.org/project/pycurl/#files
[4]: https://github.com/pycurl/pycurl/issues/439