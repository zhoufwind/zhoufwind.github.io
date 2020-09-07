---
layout: post
title: "zookeeper源码编译（mvn）"
date: 2020-01-13 05:51PM
catalog: true
tags:
    - 开发
    - ELK
    - Git
    - IDEA
---

## 下载源码

Github下载es源码[link](https://github.com/elastic/elasticsearch)：
下载后checkout到5.6分支。

## 编译、打包、运行

### java版本

java13太高，需降级到1.8，修改环境变量即可。

```bash
vi ~/.bash_profile
...
export JAVA_HOME="/Library/Java/JavaVirtualMachines/jdk1.8.0_221.jdk/Contents/Home"
export CLASSPATH="$JAVA_HOME/lib"
```

确认java版本：

```bash
$ java -version
java version "1.8.0_221"
Java(TM) SE Runtime Environment (build 1.8.0_221-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.221-b11, mixed mode)
```

### gradle版本降级：

参照官网升级文档：[Gradle | Installation | Installing manually](https://gradle.org/install/#manually)

```bash
sudo mkdir /opt/gradle
sudo chown zhoufeng12:staff /opt/gradle
unzip -d /opt/gradle ~/Downloads/gradle-4.6-bin.zip
which gradle
ll /usr/local/bin/gradle
mv /usr/local/bin/gradle /usr/local/bin/gradle.bak
ln -s /opt/gradle/gradle-4.6/bin/gradle /usr/local/bin/gradle
gradle -v
```

同时再强制修改环境变量：
```bash
vi ~/.bash_profile
...
export PATH=$PATH:/opt/gradle/gradle-4.6/bin
```

确认gradle版本：

```bash
$ gradle -v

------------------------------------------------------------
Gradle 4.6
------------------------------------------------------------

Build time:   2018-02-28 13:36:36 UTC
Revision:     8fa6ce7945b640e6168488e4417f9bb96e4ab46c

Groovy:       2.4.12
Ant:          Apache Ant(TM) version 1.9.9 compiled on February 2 2017
JVM:          1.8.0_221 (Oracle Corporation 25.221-b11)
OS:           Mac OS X 10.14.6 x86_64
```

### 进行编译

1. 开始构建

```bash
$ ./gradlew assemble
...
BUILD SUCCESSFUL in 5m 8s
488 actionable tasks: 488 executed
```

2. 生成IDEA项目文件

```bash
$ gradle idea
...
BUILD SUCCESSFUL in 29s
167 actionable tasks: 167 executed
```

### 创建配置文件目录

1. 拷贝配置文件目录及文件：

```bash
$ mkdir /Users/zhoufeng12/es
$ chown -R zhoufeng12:staff /Users/zhoufeng12/es

$ cd distribution/zip/build/distributions/
$ unzip -q elasticsearch-5.6.17-SNAPSHOT.zip
$ cd elasticsearch-5.6.17-SNAPSHOT
$ cp -r config modules plugins /Users/zhoufeng12/es
```

2. 手工写入elasticsearch.policy配置：

```bash
$ vi /Users/zhoufeng12/es/config/elasticsearch.policy
grant {
    permission javax.management.MBeanTrustPermission "register";
    permission javax.management.MBeanServerPermission "createMBeanServer";
};
```

### 导入IDEA

1. gradle配置页：

![img](/img/in-post/post-200907-es-src-build/71599488841_.pic_hd.jpg)

2. Run/Debug配置：

![img](/img/in-post/post-200907-es-src-build/61599488779_.pic_hd.jpg)

VM中输入以下配置：

```
-Des.path.home=/Users/zhoufeng12/es
-Des.path.conf=/Users/zhoufeng12/es/config
-Xms1g
-Xmx1g
-Dlog4j2.disable.jmx=true
-Djava.security.policy=/Users/zhoufeng12/es/config/elasticsearch.policy
```

### 开始运行

点击debug按钮，开始启动：

```bash
10:12:29 PM: Executing task 'Elasticsearch.main()'...

Starting Gradle Daemon...
Connected to the target VM, address: '127.0.0.1:52551', transport: 'socket'
Gradle Daemon started in 1 s 244 ms
:buildSrc:compileJava UP-TO-DATE
:buildSrc:compileGroovy UP-TO-DATE
:buildSrc:writeVersionProperties UP-TO-DATE
:buildSrc:processResources UP-TO-DATE
:buildSrc:classes UP-TO-DATE
:buildSrc:jar UP-TO-DATE
:buildSrc:assemble UP-TO-DATE
:buildSrc:compileTestJava UP-TO-DATE
:buildSrc:compileTestGroovy NO-SOURCE
:buildSrc:processTestResources NO-SOURCE
:buildSrc:testClasses UP-TO-DATE
:buildSrc:test NO-SOURCE
:buildSrc:check UP-TO-DATE
:buildSrc:build UP-TO-DATE
=======================================
Elasticsearch Build Hamster says Hello!
=======================================
  Gradle Version        : 4.6
  OS Info               : Mac OS X 10.14.6 (x86_64)
  JDK Version           : Oracle Corporation 1.8.0_221 [Java HotSpot(TM) 64-Bit Server VM 25.221-b11]
  JAVA_HOME             : /Library/Java/JavaVirtualMachines/jdk1.8.0_221.jdk/Contents/Home
:core:compileJavaNote: Some input files use or override a deprecated API.
Note: Recompile with -Xlint:deprecation for details.
Note: Some input files use unchecked or unsafe operations.
Note: Recompile with -Xlint:unchecked for details.

:core:generateModulesList
:core:generatePluginsList
:core:processResources
:core:classes
:core:Elasticsearch.main()Connected to the VM started by ':core:Elasticsearch.main()' (localhost:52579). Open the debugger session tab

[2020-09-07T22:13:16,329][INFO ][o.e.n.Node               ] [] initializing ...
[2020-09-07T22:13:16,466][INFO ][o.e.e.NodeEnvironment    ] [De2_4me] using [1] data paths, mounts [[/ (/dev/disk1s1)]], net usable_space [78.4gb], net total_space [232.4gb], spins? [unknown], types [apfs]
[2020-09-07T22:13:16,466][INFO ][o.e.e.NodeEnvironment    ] [De2_4me] heap size [981.5mb], compressed ordinary object pointers [true]
[2020-09-07T22:13:16,468][INFO ][o.e.n.Node               ] node name [De2_4me] derived from node ID [De2_4meuSCKGhuI49kT86A]; set [node.name] to override
[2020-09-07T22:13:16,468][INFO ][o.e.n.Node               ] version[5.6.17-SNAPSHOT], pid[64570], build[Unknown/Unknown], OS[Mac OS X/10.14.6/x86_64], JVM[Oracle Corporation/Java HotSpot(TM) 64-Bit Server VM/1.8.0_221/25.221-b11]
[2020-09-07T22:13:16,468][INFO ][o.e.n.Node               ] JVM arguments [-Des.path.conf=/Users/zhoufeng12/es/config, -Des.path.home=/Users/zhoufeng12/es, -Djava.security.policy=/Users/zhoufeng12/es/config/elasticsearch.policy, -Dlog4j2.disable.jmx=true, -agentlib:jdwp=transport=dt_socket,server=n,suspend=y,address=52579, -Xms1g, -Xmx1g, -Dfile.encoding=UTF-8, -Duser.country=CN, -Duser.language=en, -Duser.variant]
[2020-09-07T22:13:16,469][WARN ][o.e.n.Node               ] version [5.6.17-SNAPSHOT] is a pre-release version of Elasticsearch and is not suitable for production
[2020-09-07T22:13:23,937][INFO ][o.e.p.PluginsService     ] [De2_4me] loaded module [aggs-matrix-stats]
[2020-09-07T22:13:23,937][INFO ][o.e.p.PluginsService     ] [De2_4me] loaded module [ingest-common]
[2020-09-07T22:13:23,937][INFO ][o.e.p.PluginsService     ] [De2_4me] loaded module [lang-expression]
[2020-09-07T22:13:23,937][INFO ][o.e.p.PluginsService     ] [De2_4me] loaded module [lang-groovy]
[2020-09-07T22:13:23,937][INFO ][o.e.p.PluginsService     ] [De2_4me] loaded module [lang-mustache]
[2020-09-07T22:13:23,937][INFO ][o.e.p.PluginsService     ] [De2_4me] loaded module [lang-painless]
[2020-09-07T22:13:23,937][INFO ][o.e.p.PluginsService     ] [De2_4me] loaded module [parent-join]
[2020-09-07T22:13:23,938][INFO ][o.e.p.PluginsService     ] [De2_4me] loaded module [percolator]
[2020-09-07T22:13:23,938][INFO ][o.e.p.PluginsService     ] [De2_4me] loaded module [reindex]
[2020-09-07T22:13:23,938][INFO ][o.e.p.PluginsService     ] [De2_4me] loaded module [transport-netty3]
[2020-09-07T22:13:23,938][INFO ][o.e.p.PluginsService     ] [De2_4me] loaded module [transport-netty4]
[2020-09-07T22:13:23,938][INFO ][o.e.p.PluginsService     ] [De2_4me] no plugins loaded
[2020-09-07T22:13:26,283][INFO ][o.e.d.DiscoveryModule    ] [De2_4me] using discovery type [zen]
[2020-09-07T22:13:26,838][INFO ][o.e.n.Node               ] initialized
[2020-09-07T22:13:26,839][INFO ][o.e.n.Node               ] [De2_4me] starting ...
[2020-09-07T22:13:26,917][INFO ][i.n.u.i.PlatformDependent] Your platform does not provide complete low-level API for accessing direct buffers reliably. Unless explicitly requested, heap buffer will always be preferred to avoid potential system instability.
[2020-09-07T22:13:27,144][INFO ][o.e.t.TransportService   ] [De2_4me] publish_address {127.0.0.1:9300}, bound_addresses {[::1]:9300}, {127.0.0.1:9300}
[2020-09-07T22:13:30,219][INFO ][o.e.c.s.ClusterService   ] [De2_4me] new_master {De2_4me}{De2_4meuSCKGhuI49kT86A}{D9xl21kRSOGoeOvuKoN25w}{127.0.0.1}{127.0.0.1:9300}, reason: zen-disco-elected-as-master ([0] nodes joined)
[2020-09-07T22:13:30,250][INFO ][o.e.h.n.Netty4HttpServerTransport] [De2_4me] publish_address {127.0.0.1:9200}, bound_addresses {[::1]:9200}, {127.0.0.1:9200}
[2020-09-07T22:13:30,250][INFO ][o.e.n.Node               ] [De2_4me] started
[2020-09-07T22:13:30,250][INFO ][o.e.g.GatewayService     ] [De2_4me] recovered [0] indices into cluster_state
```

![img](/img/in-post/post-200907-es-src-build/81599489280_.pic.jpg)
