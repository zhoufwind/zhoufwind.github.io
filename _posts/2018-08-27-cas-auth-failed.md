---
layout: post
title: "CAS单点登录异常问题排查记录"
subtitle: "先解决问题，再填坑"
date: 2018-08-27 06:14PM
catalog: true
tags:
    - 运维
    - CAS
---

### 背景

快下班的时候，接到北研的同事反馈CAS登录异常的反馈，所有用到单点登录的场景，点击登录后均跳500错误。

![img](/img/in-post/post-180827-cas-auth-failed/WechatIMG2045.png)

![img](/img/in-post/post-180827-cas-auth-failed/WechatIMG2044.png)

### 问题

自己尝试重现了下，但无法重现，同时问了下北研其他同学，单点登录也OK，说明不是大范围异常，属个别用户问题。

看到ticket过期提示，以为是session相关问题，建议用户清楚浏览器缓存、换浏览器访问、重启电脑、换别人电脑等等方式，最后尝试均告失败。

由此看来应该不是客户端问题，需要上服务器查看服务器日志了。

登上CAS服务器，先查CAS程序日志，并未发现与该用户异常相关日志，仅包含一些定时任务执行异常的错误，每6小时一次，一天4次：

```
2018-08-04 00:01:02,525 - crontab.py - DEBUG - 雇员信息数据同步开始...
2018-08-04 00:01:02,525 - crontab.py - DEBUG - 开始从人事系统获取内部雇员(即有LDAP账号的雇员,包括全职、外包、实习、派遣)信息...
2018-08-04 00:01:02,955 - crontab.py - ERROR - 雇员信息数据同步异常,异常信息如下:
Traceback (most recent call last):
  File "/home/hjadminops/cas_project/cas_server/crontab.py", line 96, in emp_sync
    syncer.sync()
  File "/home/hjadminops/cas_project/cas_server/crontab.py", line 82, in sync
    emps = response.json()['Data']
  File "/usr/local/lib/python3.4/site-packages/requests/models.py", line 826, in json
    return complexjson.loads(self.text, **kwargs)
  File "/usr/local/lib/python3.4/site-packages/simplejson/__init__.py", line 516, in loads
    return _default_decoder.decode(s)
  File "/usr/local/lib/python3.4/site-packages/simplejson/decoder.py", line 370, in decode
    obj, end = self.raw_decode(s)
  File "/usr/local/lib/python3.4/site-packages/simplejson/decoder.py", line 400, in raw_decode
    return self.scan_once(s, idx=_w(s, idx).end())
simplejson.scanner.JSONDecodeError: Expecting value: line 3 column 1 (char 4)
```

然后查看nginx日志，发现500响应日志，但并没有异常原因：

![img](/img/in-post/post-180827-cas-auth-failed/WechatIMG2046.png)

最后查看uwsgi日志，这才发现异常原因：

```
2018-08-27 18:10:39,187 [DEBUG] b'uWSGIWorker8Core1':140265504601856 ./cas_server/views.py:1703 - require push auth code from <client_ip>
[pid: 27855|app: 0|req: 2/2] <client_ip2> () {46 vars in 988 bytes} [Mon Aug 27 18:10:39 2018] GET /cas/push_auth_code?username=<domain_name>&randomint=<randomint> => generated 1 bytes in 233 msecs (HTTP/1.1 200) 4 headers in 146 bytes (1 switches on core 1)
2018-08-27 18:10:43,929 [INFO] b'uWSGIWorker7Core0':140265699432384 ./cas_server/views.py:533 - valid auth code params: {'timestamp': 1535364643, 'user_ip': '<client_ip>', 'sign': '<sign_code>', 'key': '<key>', 'user_name': '<domain_name>', 'auth_code': '<auth_code>'}
2018-08-27 18:10:43,929 [INFO] b'uWSGIWorker7Core0':140265699432384 ./cas_server/views.py:535 - valid auth from CDN
2018-08-27 18:10:43,974 [INFO] b'uWSGIWorker7Core0':140265699432384 ./cas_server/views.py:787 - User <domain_name> successfully authenticated
2018-08-27 18:10:44,387 [ERROR] b'uWSGIWorker7Core0':140265699432384 ./cas_server/middleware/casMiddleWare.py:6 - unhandle exception happen
Traceback (most recent call last):
  File "/usr/local/lib/python3.4/site-packages/django/core/handlers/base.py", line 147, in get_response
    response = wrapped_callback(request, *callback_args, **callback_kwargs)
  File "/usr/local/lib/python3.4/site-packages/django/views/decorators/debug.py", line 76, in sensitive_post_parameters_wrapper
    return view(request, *args, **kwargs)
  File "/usr/local/lib/python3.4/site-packages/django/views/generic/base.py", line 68, in view
    return self.dispatch(request, *args, **kwargs)
  File "/usr/local/lib/python3.4/site-packages/django/views/generic/base.py", line 88, in dispatch
    return handler(request, *args, **kwargs)
  File "./cas_server/views.py", line 712, in post
    response = self.common()
  File "./cas_server/views.py", line 1154, in common
    return self.authenticated()
  File "./cas_server/views.py", line 1024, in authenticated
    return self.service_login()
  File "./cas_server/views.py", line 936, in service_login
    renew=self.renewed
  File "./cas_server/models.py", line 395, in get_service_url
    ticket = self.get_ticket(ServiceTicket, service, service_pattern, renew)
  File "./cas_server/models.py", line 370, in get_ticket
    attributs=self.attributs,
  File "./cas_server/models.py", line 294, in attributs
    return utils.import_attr(settings.CAS_AUTH_CLASS)(self.username, isTestPassword=False).attributs()
  File "./cas_server/auth.py", line 341, in __init__
    fail_silently=False
  File "/usr/local/lib/python3.4/site-packages/django/core/mail/__init__.py", line 61, in send_mail
    return mail.send()
  File "/usr/local/lib/python3.4/site-packages/django/core/mail/message.py", line 292, in send
    return self.get_connection(fail_silently).send_messages([self])
  File "/usr/local/lib/python3.4/site-packages/django/core/mail/backends/smtp.py", line 100, in send_messages
    new_conn_created = self.open()
  File "/usr/local/lib/python3.4/site-packages/django/core/mail/backends/smtp.py", line 67, in open
    self.connection.login(self.username, self.password)
  File "/usr/local/lib/python3.4/smtplib.py", line 652, in login
    raise SMTPAuthenticationError(code, resp)
smtplib.SMTPAuthenticationError: (535, b'5.7.8 Error: authentication failed: authentication failure')
[pid: 27852|app: 0|req: 1/3] <client_ip3> () {54 vars in 1273 bytes} [Mon Aug 27 18:10:43 2018] POST /cas/login?service=http%3A%2F%2F<dst_domain>%2Fsite%2Flogin => generated 27 bytes in 736 msecs (HTTP/1.1 500) 4 headers in 150 bytes (1 switches on core 0)
```

从这段日志可以发现是`./cas_server/auth.py`这段程序中`fail_silently`这个值异常导致，查看其源码如下：

```
            # 检查用户ldap的ou或cn是否有新增,有新增则发邮件通知
            user_ou_cns = ','.join(set((','.join(self.user['distinguishedName']) + ',' + ','.join(self.user['memberOf'])).split(',')))
            lobjects = LdapUser.objects.filter(name=self.username)
            if lobjects:
                lobject = lobjects[0]
                new_oucns = set(user_ou_cns.split(',')) - set(lobject.oucns.split(','))
                if new_oucns:
                    send_mail(
                        '生产LDAP用户OU,CN变更通知',
                        '用户:'+self.username+',新增OU,CN:'+str(new_oucns),
                        settings.SERVER_EMAIL,
                        settings.LDAP_OU_CN_CHANGE_NOTICE_EMAIL,
                        fail_silently=False
                    )
                    lobject.oucns = user_ou_cns
                    lobject.save()
            else:
                LdapUser.objects.create(name=self.username, oucns=user_ou_cns)
```

查看`cas_project/settings.py`中`SERVER_EMAIL`及`LDAP_OU_CN_CHANGE_NOTICE_EMAIL`配置，同时确认公司邮件服务器正在迁移，迁移后邮件服务器配置确有调整，所以才会导致这次500异常。

### 缓解

定位到问题后，这边首先调整`SERVER_EMAIL`及`LDAP_OU_CN_CHANGE_NOTICE_EMAIL`配置，但修改后用户反馈仍然异常。

后面考虑到这块邮件发送功能并非核心功能，注释掉这块逻辑并不会对CAS核心功能造成影响，而且应该可以快速修复问题。

于是注释掉了send_mail()该代码块后，问题得到缓解。

### 后记

这边选择了暂时忽略这个问题，等以后如果又有邮件发送方面的需求，这块还得继续排查（挖了个坑），暂时先选择优先缓解问题。