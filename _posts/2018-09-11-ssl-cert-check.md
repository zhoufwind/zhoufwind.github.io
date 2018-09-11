---
layout: post
title: "GO语言高并发模式在监控的使用"
date: 2018-09-11 03:01PM
catalog: true
tags:
    - 运维
    - HTTPS
    - SSL
---

# GO语言高并发模式在监控的使用

提到GO语言，各位朋友可能和我一样，想到是GO语言具有强大的并发处理能力。但是具体怎么实现GO的高并发呢，怎么控制并发，防止并发过度呢？下面我和大家分享一种高效，并发度可控的并发结构。为了能够更好说明，文中以并发获取域名证书的过期时间为例。

## 程序结构

![img](/img/in-post/post-180911-ssl-cert-check/WechatIMG2949.png)

这种结构分为了四个模块，分别是数据读入模块、证书时间获取模块、数据写出模块、程序调度模块。

1. 数据读入模块：用来从文件中读取被检测的域名，读入的数据写入到rc缓冲队列当中。
2. 证书时间获取模块，从rc队列中读取信息，并获取域名的证书信息，获取到的信息写入wc缓冲队列。
3. 数据写出模块，从rc缓冲队列中读取域名的证书信息并把数据写出到数据库，或者直接打印出来。
4. Main调度模块，主要是并发度的控制，前期数据准备等工作

这样三个模块分别通过缓冲区rc，wc交换数据，这样有生产者和消费者，数据处理速度不同的子程序可以高效协作。这里channel的长度暂时设置为300.

咱们先来写一下框架代码，具体各部分代码再一步一步的实现。

```go
package main

type SSLProcess struct {
	rc       chan string //读取channel
	wc       chan string //写出channel
	txtLines []string //暂存txt文档中一行一行的域名
	wg       sync.WaitGroup //用来同步所有协程完成的状态
}

func (self *SSLProcess) ReadFromFile() {
	//读取模块
}

func (self *SSLProcess) Process() {
	//获取证书时间的模块
}

func (self *SSLProcess) WriteOut() {
	//结果写出模块
}

func main() {
    //主处理调度模块

}
```

## 读取模块

读入模块主要包含两个功能：

1. 从txtLines数组中读出域名信息
2. 把读取的逐行写入到rc这个channel中，供给证书获取模块

具体的代码：

```go
func (self *SSLProcess) Read() {
	//读取模块
	for _, v := range self.txtLines {
		self.rc <- v //逐行写入到rc channel中
	}
}
```

## 证书信息获取模块

这部分的主要功能是：

1. 从rc缓冲channel中读入域名信息。
2. 调用cert这个模块，获取到相应的域名的信息。
3. 结果写入到wc缓冲channel中，供写出模块使用。

※cert部分的代码也一起打包放在文档中

具体的代码：

```go
func (self *SSLProcess) Process() {
	//获取证书时间的模块
	for v := range self.rc { //等待rc中有数据，并开始获取证书信息
		//v=www.hujiang.com hujiang.com
		d_ip := strings.Split(v, "	") //空格作为分割符
		domain := strings.TrimSpace(d_ip[0])
		server := strings.TrimSpace(d_ip[1])
		certs := cert.NewCert(server+":443", domain) //调用cert库获取证书信息
		//写入到wc缓冲channel
		self.wc <- "DOMIAN：" + domain + "|SERVER:" + server + "|过期时间：" + string(certs.NotAfter)
	}
}
```

## writeout部分的代码

写出模块主要是从wc缓冲channel中读读入，并写到数据库中，此处我们直接打印出来。

具体的代码：

```go
func (self *SSLProcess) WriteOut() {
	//结果写出模块
	for v := range self.wc { //等待wc中有数据，并打印出来
		fmt.Println(v)
		self.wg.Done()
	}
}
```

## 主函数部分

main函数这里主要实现以下几个功能：

1. 初始化结构体变量
2. 从文件中读入域名信息，并添加到txtLines数组中
3. 开启多个go协程获取证书信息
4. 调用wait，等到所有的协程处理完成

首先我们来看下输入文件的内容格式，这里的文件名字是ssl_log.txt，里面的内容是这种格式，每行一个元素，中间用空格分割，前面的是域名，后面是配置域名证书的服务器地址（有些是ip，有些是域名）：

```bash
$ cat ssl_log.txt
www.hujiang.com hujiang.com
www.cctalk.com cctalk.com
... ...
... ...
```

main函数和文件读取模块代码

```go
func (self *SSLProcess) ReadFromFile(filename string) {
	txtBytes, err := ioutil.ReadFile(filename)
	if err != nil {
		panic(fmt.Sprintf("error:%s", err.Error()))
	}

	txtContent := string(txtBytes) //文件的所有内容读入到txtContent
	lines := strings.Split(txtContent, "\n")
	for _, v := range lines {
		d_ip := strings.Split(v, "	") //空格作为分割符
		if (len(d_ip)) >= 2 {
			self.txtLines = append(self.txtLines, v)
		}
	}

}

func main() {
	//主处理模块
	sp := SSLProcess{
		rc:       make(chan string, 300), //缓冲区长度300
		wc:       make(chan string, 300), //缓冲区长度300
		txtLines: []string{},
		wg:       sync.WaitGroup{},
	}

	sp.ReadFromFile("ssl_log.txt")

	sp.wg.Add(len(sp.txtLines))

	go sp.Read()               //单个协程
	for i := 0; i < 100; i++ { //这部分比较耗时，开启多个协程处理
		go sp.Process()
	}
	go sp.WriteOut() //单个协程

	sp.wg.Wait() //等待所有的任务完成

}
```

## 总结

效果检验，在3000多个域名的情况，12核心机器用这种模式可以在4秒以内跑完，原来的方式大概需要20多分钟跑完，速度提升了很多。

程序在主程序部分通过有缓冲的channel和控制go 协程的数量，有效的控制了并发的数量，防止过度的并发，反而拖慢整体的速度。在实际的使用中，输入端可以是tcp端口接收到的数据，或者是从滚动的日志文件中读取，还可以通过http接口接收数据。在使用有阻塞的监听端口的输入数据的方式时候，大家可以不使用sync.WaitGroup。

最后附上main完整的代码：

```go
// sslcheck project main.go
package main

import (
	"fmt"
	"io/ioutil"
	"sslcheck/cert"
	"strings"
	"sync"
)

type SSLProcess struct {
	rc       chan string    //读取channel
	wc       chan string    //写出channel
	txtLines []string       //暂存txt文档中一行一行的域名
	wg       sync.WaitGroup //用来同步所有协程完成的状态
}

func (self *SSLProcess) Read() {
	//读取模块
	for _, v := range self.txtLines {
		self.rc <- v //逐行写入到rc channel中
	}
}

func (self *SSLProcess) Process() {
	//获取证书时间的模块
	for v := range self.rc { //等待rc中有数据，并开始获取证书信息
		//v=www.hujiang.com hujiang.com
		d_ip := strings.Split(v, "	") //空格作为分割符
		domain := strings.TrimSpace(d_ip[0])
		server := strings.TrimSpace(d_ip[1])
		certs := cert.NewCert(server+":443", domain) //调用cert库获取证书信息
		//写入到wc缓冲channel
		self.wc <- "DOMIAN：" + domain + "|SERVER:" + server + "|过期时间：" + string(certs.NotAfter)
	}
}

func (self *SSLProcess) WriteOut() {
	//结果写出模块
	for v := range self.wc { //等待wc中有数据，并打印出来
		fmt.Println(v)
		self.wg.Done()
	}
}

func (self *SSLProcess) ReadFromFile(filename string) {
	txtBytes, err := ioutil.ReadFile(filename)
	if err != nil {
		panic(fmt.Sprintf("error:%s", err.Error()))
	}

	txtContent := string(txtBytes) //文件的所有内容读入到txtContent
	lines := strings.Split(txtContent, "\n")
	for _, v := range lines {
		d_ip := strings.Split(v, "	") //空格作为分割符
		if (len(d_ip)) >= 2 {
			self.txtLines = append(self.txtLines, v)
		}
	}

}

func main() {
	//主处理模块
	sp := SSLProcess{
		rc:       make(chan string, 300), //缓冲区长度300
		wc:       make(chan string, 300), //缓冲区长度300
		txtLines: []string{},
		wg:       sync.WaitGroup{},
	}

	sp.ReadFromFile("ssl_dns.txt")

	sp.wg.Add(len(sp.txtLines))

	go sp.Read()               //单个协程
	for i := 0; i < 200; i++ { //这部分比较耗时，开启多个协程处理
		go sp.Process()
	}
	go sp.WriteOut() //单个协程

	sp.wg.Wait() //等待所有的任务完成

}

```

项目的所有代码：关注公众号 发送sslcheck获取下载



