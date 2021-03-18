+++
tags = ["golang", "nsq", "消息队列"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang遇上NSQ消息队列"
date = "2020-07-14T22:00:38+08:00"
+++


# 简介

NSQ是一个基于Go语言的分布式实时消息平台, 它具有分布式、去中心化的拓扑结构，支持无限水平扩展。无单点故障、故障容错、高可用性以及能够保证消息的可靠传递的特征。另外，NSQ非常容易配置和部署, 且支持众多的消息协议。支持多种客户端，协议简单。

# NSQ的几个组件
 * nsqd：一个负责接收、排队、转发消息到客户端的守护进程
 * nsqlookupd：管理拓扑信息, 用于收集nsqd上报的topic和channel,并提供最终一致性的发现服务的守护进程
 * nsqadmin：一套Web用户界面，可实时查看集群的统计数据和执行相应的管理任务

# Docker安装

## 搭建主NSQ服务

### 获取到自己的服务器ip

我这里就是我服务器的外网ip
```39.106.33.33```

### 获取镜像

```docker
docker pull nsqio/nsq  #拉取nsq镜像
docker images          #查看nsq镜像

```

### 启动nsqlookupd服务
这个服务就是监控所有的nsq节点服务，这里开了两个端口4160和4161，4160就是来给节点访问的，4161是为了nsqadmin使用

```docker
docker run -d --name lookupd -p 4160:4160 -p 4161:4161 nsqio/nsq:latest /nsqlookupd

docker exec -ti lookupd /bin/sh    #进入容器，查看nsq目录结构

```

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/14/20200714221747.png)

### 启动nsqadmin管理系统

```docker
docker run -d --name nsqadmin 
    -p 4171:4171 nsqio/nsq /nsqadmin 
    --lookupd-http-address=第一步获取的服务器ip:4161

```

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/14/20200714221844.png)

## 部署NSQd节点服务

### 在主服务器上开启一个nsqd节点服务

```docker
docker run -d --name nsqd -p 4150:4150 -p 
  4151:4151 nsqio/nsq:latest /nsqd 
  --broadcast-address=当前服务器ip 
  --lookupd-tcp-address=第一步获取的服务器ip:4160

```

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/14/20200714221947.png)


### 创建从服务器（可以省略，根据需求来）

启动一个nsqd服务

```docker
docker run -d --name nsqd -p 4150:4150 -p 
    4151:4151 nsqio/nsq:latest 
    /nsqd 
    --broadcast-address=当前服务器ip 
    --lookupd-tcp-address=主服务器ip:4160

```
## 进入后台

访问：ip:4171

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/14/20200714222423.png)

搭建成功


# Golang操作使用nsq

## 安装依赖
```golang
"github.com/nsqio/go-nsq"

```

## 服务端(消费者)
```golang
package main

import (
	"encoding/json"
	"fmt"
	"github.com/nsqio/go-nsq"
	"sync"
	"time"
)

var (
tcpNsqdAddrr = "xxx.xxx.xxx.xxx:4150"
)

//声明一个结构体，实现HandleMessage接口方法（根据文档的要求）
type NsqHandler struct{}

//实现HandleMessage方法
//message是接收到的消息
func (s *NsqHandler) HandleMessage(message *nsq.Message) error {
	//打印消息的一些基本信息
	fmt.Printf("msg.Timestamp="+
		"%v, msg.nsqaddress="+
		"%s,msg.body="+
		"%s \n",
		time.Unix(0, message.Timestamp).Format("2006-01-02 03:04:05"),
		message.NSQDAddress,
		string(message.Body))
	//解析传递的json数据
    var mapData map[string]interface{}
    _ = json.Unmarshal(message.Body, &mapData)
    //具体的业务逻辑
	return nil
}

func main() {

	//初始化配置
	config := nsq.NewConfig()

	//创造消费者，参数一时订阅的主题，参数二是使用的通道
	com, err := nsq.NewConsumer("wpan", "email", config)
	if err != nil {
		fmt.Println(err)
	}

	//添加处理回调
	com.AddHandler(&NsqHandler{})

	//连接对应的nsqd
	err = com.ConnectToNSQD(tcpNsqdAddrr)

	if err != nil {
		fmt.Println(err)
	}

	//只是为了不结束此进程，这里没有意义
	var wg = &sync.WaitGroup{}
	wg.Add(1)
	wg.Wait()

}

```

## 客户端(生产者)

```golang
package main

import (
	"encoding/json"
	"fmt"
	"github.com/nsqio/go-nsq"
)

var (
	//nsqd的地址，使用了tcp监听的端口
	tcpNsqdAddrr = "xxx.xxx.xxx.xxx:4150"
)

func main() {
	//初始化配置
	config := nsq.NewConfig()

	for i := 0; i < 100; i++ {
		//创建100个生产者
		tPro, err := nsq.NewProducer(tcpNsqdAddrr, config)
		if err != nil {
			fmt.Println("创建生产者", err)
		}
		//主题
		topic := "Insert"
    //主题内容
    //封装发送数据
        Command := make(map[string]interface{})
    
		data, err := json.Marshal(Command)

		//发布消息
		err = tPro.Publish(topic, []byte(data))
		if err != nil {
			fmt.Println("发布消息", err)
		}
	}
}

```

参考连接：[https://nsq.io/overview/quick_start.html](https://nsq.io/overview/quick_start.html)

参考连接：[https://github.com/nsqio/go-nsq](https://github.com/nsqio/go-nsq)




