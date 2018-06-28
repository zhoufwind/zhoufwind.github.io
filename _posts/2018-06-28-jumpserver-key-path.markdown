---
layout: post
title: JumpServer故障排查实例
date: 2018-06-28 22:29:00 +0800
img: 15301714176412.png
tags:
  - 运维
  - JumpServer
  - 源码分析
---
### 背景
公司之前开发及维护JumpServer的同学离职了，维护工作转交到了我这边。领导要求运维这边根据遗留下来的源码，重新部署一套，以保证对JumpServer的支持能力。

本周完成了迁移工作，但不清楚是否会有一些自己没注意到的暗坑，心里也不是很稳，这不，迁移后第二天，反馈就来了。

### 问题
接到反馈，新推送的系统账户推送失败，用户登录直接报超时，随后退出。

立即上服务器查看日志，报错信息如下：
```
2018-06-28 14:08:21,668 - connect.py - DEBUG - Traceback (most recent call last):
  File "/opt/jumpserver/connect.py", line 947, in <module>
    main()
  File "/opt/jumpserver/connect.py", line 909, in main
    nav.try_connect()
  File "/opt/jumpserver/connect.py", line 653, in try_connect
    ssh_tty.connect()
  File "/opt/jumpserver/connect.py", line 435, in connect
    ssh = self.get_connection()
  File "/opt/jumpserver/connect.py", line 241, in get_connection
    connect_info = self.get_connect_info()
  File "/opt/jumpserver/connect.py", line 229, in get_connect_info
    role_key = get_role_key(self.user, self.role)  # 获取角色的key，因为ansible需要权限是600，所以统一生成用户_角色key
  File "/opt/jumpserver/jumpserver/api.py", line 96, in get_role_key
    with open(os.path.join(role.key_path, 'id_rsa')) as fk:
IOError: [Errno 2] No such file or directory: u'/home/optdir/jumpserver/keys/role_key/key-d92xxxbad/id_rsa'
```

超时的原因十分明显，读取key文件失败，读key的程序目录是`/home/optdir/jumpserver/`，而这个目录是我迁移之前的目录，迁移之后，程序目录是`/opt/jumpserver`。

### 排查
为啥程序路径被缓存住了呢？

源码搜jumpserver/api.py这端代码，找到get_role_key这个方法：
```
def get_role_key(user, role):
    """
    由于role的key的权限是所有人可以读的， ansible执行命令等要求为600，所以拷贝一份到特殊目录
    :param user:
    :param role:
    :return: self key path
    """
    user_role_key_dir = os.path.join(KEY_DIR, 'user')
    user_role_key_path = os.path.join(user_role_key_dir, '%s_%s.pem' % (user.username, role.name))
    mkdir(user_role_key_dir, mode=777)
    if not os.path.isfile(user_role_key_path):
        with open(os.path.join(role.key_path, 'id_rsa')) as fk:
            with open(user_role_key_path, 'w') as fu:
                fu.write(fk.read())
        logger.debug(u"创建新的系统用户key %s, Owner: %s" % (user_role_key_path, user.username))
        chown(user_role_key_path, user.username)
        os.chmod(user_role_key_path, 0600)
    return user_role_key_path
```
可以看到是在读取`role.key_path`出现了问题，我们再搜`key_path`，发现`PermRole`这个库中有`key_path`这个值：
```
class PermRole(models.Model):
    name = models.CharField(max_length=100, unique=True)
    comment = models.CharField(max_length=100, null=True, blank=True, default='')
    password = models.CharField(max_length=512)
    key_path = models.CharField(max_length=100)
    date_added = models.DateTimeField(auto_now=True)
    sudo = models.ManyToManyField(PermSudo, related_name='perm_role')

    def __unicode__(self):
        return self.name
```
看到Model，看来涉及到DB了，我们再查了下DB，果然发现了问题：
```
mysql> select * from jperm_permrole where key_path LIKE '%d92xxxbad%'\G
*************************** 1. row ***************************
        id: 34
      name: devuser
   comment: 开发人员使用的普通用户
  password: 161xxx377
  key_path: /home/optdir/jumpserver/keys/role_key/key-d92xxxbad
date_added: 2017-06-19 12:36:58
1 row in set (0.00 sec)

```
到这边已经很清晰了，程序把统账户的key信息入DB库了，而不是根据真实程序路劲下key文件的相对路径进行读取，从而造成了异常。

### 缓解
如果要改DB数据怕会改出新BUG，这边取了一个讨巧的办法，执行了一条命令就暂时缓解了问题：
```
# ln -s /opt/jumpserver /home/optdir/jumpserver
```

### 后记
做程序迁移，最好和原程序保持一致，甚至是文件路径，否则说不定就会踩到雷了。