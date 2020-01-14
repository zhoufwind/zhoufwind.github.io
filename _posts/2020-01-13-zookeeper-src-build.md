---
layout: post
title: "zookeeper源码编译（mvn）"
date: 2020-01-13 05:51PM
catalog: true
tags:
    - 开发
    - Zookeeper
    - Git
    - IDEA
---

### 背景

老版的zk是通过ant进行编译的，但最新的zk源码中已经没了`build.xml`，而多了`pom.xml`，也就是说构建方式由原先的Ant变成了Maven，源码下下来后，直接编译、运行是跑不起来的，有一些配置需要调整，这边做下总结。

### 注意点

#### 导入项目类型

导入类型是`Maven`，不要再选`Eclipse`了。

#### 构建项目

点击IDEA最右侧的“Maven”，点击第二个小图标：“Generate Sources and Update Folders For All Projects”。

![img](/img/in-post/post-200113-zk-src-build/WechatIMG10.png)

#### 修改jre

打开项目结构（⌘ + ;），指定jdk为`1.8`。

#### 修改gitignore文件

由于开发环境需要进行调试，临时配置文件是不需要进代码仓库的，所以进行排除。

```
# Ignore debug conf
conf/zoo.cfg
conf/zoo*.cfg
zookeeper-server/src/main/resources/log4j.properties
```

#### 拷贝配置文件

  1. 拷贝`zoo_sample.cfg`文件至相同文件夹下，名为：`zoo.cfg`，配置全部使用默认；
  2. 创建`/tmp/zookeeper`目录，用于存放zk数据；
  3. 拷贝`log4j.properties`文件至：`zookeeper-server/src/main/resources`，文件名还是`log4j.properties`不变；
  4. 打开项目结构，将`resources`目录标记为`Resources`类型（不配置的话日志配置不生效）；
    ![img](/img/in-post/post-200113-zk-src-build/WechatIMG22.png)

#### 增加运行启动项

配置启动项：

```
QuorumPeerMain
org.apache.zookeeper.server.quorum.QuorumPeerMain
conf/zoo.cfg
```

![img](/img/in-post/post-200113-zk-src-build/WechatIMG14.png)

#### 增加Info类

默认如果不添加`Info`类，编译不通过：
![img](/img/in-post/post-200113-zk-src-build/WechatIMG16.png)

创建Info文件，路径：`src/main/java/org/apache/zookeeper/version/Info.java`

```java
public interface Info {
    public static final int MAJOR=3;
    public static final int MINOR=4;
    public static final int MICRO=6;
    public static final String QUALIFIER=null;
    public static final int REVISION=-1;
    public static final String REVISION_HASH = "1";
    public static final String BUILD_DATE="12/25/2019 09:55 GMT";
}
```

#### 修改pom.xml

默认的`pom.xml`有问题，直接编译多少都会报找不到类的错，需要手工调整：

- java.lang.ClassNotFoundException: com.codahale.metrics.Reservoir
  ```xml
  <dependency>
      <groupId>io.dropwizard.metrics</groupId>
      <artifactId>metrics-core</artifactId>
      <version>3.1.0</version>
  </dependency>
  ```

- java.lang.ClassNotFoundException: org.xerial.snappy.SnappyInputStream
  ```xml
  <dependency>
    <groupId>org.xerial.snappy</groupId>
    <artifactId>snappy-java</artifactId>
    <version>1.1.7.3</version>
  </dependency>
  ```

- java.lang.ClassNotFoundException: org.eclipse.jetty.server.HttpConfiguration$Customizer
  ```xml
  <dependency>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-server</artifactId>
  </dependency>
  ```

- java.lang.ClassNotFoundException: org.eclipse.jetty.servlet.ServletContextHandler
  ```xml
  <dependency>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-servlet</artifactId>
  </dependency>
  ```

全部修改完毕后，服务端可以正常启动：
![img](/img/in-post/post-200113-zk-src-build/WechatIMG19.png)

#### 创建客户端启动项

配置启动项：

```
ZooKeeperMain
org.apache.zookeeper.ZooKeeperMain
```

![img](/img/in-post/post-200113-zk-src-build/WechatIMG18.png)

#### 修改pom.xml

- java.lang.ClassNotFoundException: org.apache.commons.cli.ParseException
  ```xml
  <dependency>
    <groupId>commons-cli</groupId>
    <artifactId>commons-cli</artifactId>
  </dependency>
  ```

全部修改完毕后，客户端可以正常启动：
![img](/img/in-post/post-200113-zk-src-build/WechatIMG20.png)

