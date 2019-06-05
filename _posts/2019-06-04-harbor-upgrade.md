---
layout: post
title: "Harbor版本升级记录（v1.5.4 -> v1.8.0）"
date: 2019-06-04 11:42AM
catalog: true
tags:
    - 运维
    - Harbor
---

### 背景

原harbor版本：`v1.5.4`  
待升级版本：`v1.8.0`

由于harbor从`v1.6.0`版本开始，后端数据库由`MariaDB`改为`Postgresql`，所以在升级过程中，必须先升级到`v1.6.0`版本，再升级至`v1.8.0`。

### 升级步骤

#### v1.5.4 -> v1.6.0

该步骤中，由于数据库变更，所以需对数据库进行迁移。另外在升级过程中会改变数据库文件（database schema）以及harbor配置文件（harbor.cfg），所以必须做好备份工作，以备进行回滚操作。

迁移步骤：

1. 关闭harbor服务：  
```
cd /usr/local/src/harbor
docker-compose down
```

2. 备份harbor程序以及数据库文件：  
```
cd ..
mv harbor harbor.bak.v1.5.4
cp -rf /data/database /data/database.bak.v1.5.4
```

3. 下载harbor v1.6.0：  
```
wget https://storage.googleapis.com/harbor-releases/release-1.6.0/harbor-offline-installer-v1.6.0.tgz
tar zxf harbor-offline-installer-v1.6.0.tgz
```

4. 下载harbor迁移工具：  
```
docker pull goharbor/harbor-migrator:v1.6.0
```

5. 创建harbor配置文件备份文件夹，并且进行备份操作：  
```
mkdir harbor.migrate.v1.5.4
docker run -it --rm -e DB_USR=root -e DB_PWD=root123 -v /data/database:/var/lib/mysql -v /usr/local/src/harbor.bak.v1.5.4/harbor.cfg:/harbor-migration/harbor-cfg/harbor.cfg -v /usr/local/src/harbor.migrate.v1.5.4:/harbor-migration/backup goharbor/harbor-migrator:v1.5.0 backup
```

6. 升级数据库，修改harbor配置文件，迁移数据：  
```
docker run -it --rm -e DB_USR=root -e DB_PWD=root123 -v /data/database:/var/lib/mysql -v /usr/local/src/harbor.bak.v1.5.4/harbor.cfg:/harbor-migration/harbor-cfg/harbor.cfg goharbor/harbor-migrator:v1.6.0 up

mkdir /data/notary-db/
docker run -it --rm -e DB_USR=root -v /data/notary-db/:/var/lib/mysql -v /data/database:/var/lib/postgresql/data goharbor/harbor-migrator:v1.6.0 --db up

mkdir /data/clair-db/
docker run -it --rm -v /data/clair-db/:/clair-db -v /data/database:/var/lib/postgresql/data goharbor/harbor-migrator:v1.6.0 --db up
```

7. 创建harbor配置文件（harbor.cfg），部署harbor应用，启动harbor：  
```
cd harbor
cp harbor.cfg harbor.cfg.src
vi harbor.cfg
mkdir cert
wget ftp://xxx:xxx@x.x.x.x/cert/_.yeshj.com.crt -P /usr/local/src/harbor/cert/
wget ftp://xxx:xxx@x.x.x.x/cert/_.yeshj.com.key -P /usr/local/src/harbor/cert/
./install.sh 
```

#### v1.6.0 -> v1.8.0

该步骤中，由于在v1.6.0版本之后，harbor会在启动服务时自行迁移数据库数据，所以无需再单独迁移数据库。

迁移步骤：

1. 关闭harbor服务：  
```
cd /usr/local/src/harbor
docker-compose down
```

2. 备份harbor程序以及数据库文件：  
```
cd ..
mv harbor harbor.bak.v1.6.0
cp -rf /data/database /data/database.bak.v1.6.0
```

3. 下载harbor v1.8.0：  
```
wget https://storage.googleapis.com/harbor-releases/release-1.8.0/harbor-offline-installer-v1.8.0.tgz
tar zxf harbor-offline-installer-v1.8.0.tgz
```

4. 下载harbor迁移工具：  
```
docker pull goharbor/harbor-migrator:v1.8.0
```

5. 升级harbor配置文件，也即`harbor.cfg`到`harbor.yml`：  
```
cd harbor
cp harbor.yml harbor.yml.src
docker run -it --rm -v /usr/local/src/harbor.bak.v1.6.0/harbor.cfg:/harbor-migration/harbor-cfg/harbor.cfg -v /usr/local/src/harbor/harbor.yml:/harbor-migration/harbor-cfg-out/harbor.yml goharbor/harbor-migrator:v1.8.0 --cfg up
```

6. 部署harbor应用，启动harbor：  
```
mkdir cert
wget ftp://xxx:xxx@x.x.x.x/cert/_.yeshj.com.crt -P /usr/local/src/harbor/cert/
wget ftp://xxx:xxx@x.x.x.x/cert/_.yeshj.com.key -P /usr/local/src/harbor/cert/
./install.sh 
```

### 参考文档

[Harbor upgrade and database migration guide(v1.6.0)](https://github.com/goharbor/harbor/blob/release-1.6.0/docs/migration_guide.md)  
[Harbor upgrade and migration guide(v1.8.0)](https://github.com/goharbor/harbor/blob/release-1.8.0/docs/migration_guide.md)

### 资源下载
- harbor-offline-installer-v1.5.4.tgz (md5: 61e8febfca5cd4e957e102b6e2fe42c0)  
链接: https://pan.baidu.com/s/16NVxp0Bu8o4Px0lGEGkVtA 提取码: qgx4

- harbor-offline-installer-v1.6.0.tgz (md5: d68d85d164020b445bcf479d17d53873)  
链接: https://pan.baidu.com/s/1ZGqeE3oYQ3eSnoHeGBYyzw 提取码: 6w5t

- harbor-offline-installer-v1.8.0.tgz (md5: f299bcfcd83bd421403113ab654890a9)  
链接: https://pan.baidu.com/s/1vbx1aNaChKbnORu4_8jJ4Q 提取码: nwr4
