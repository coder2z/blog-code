+++
tags = ["grpc", "微服务", "protobuf", "golang"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang微服务开发-grpc"
date = "2020-07-10T08:12:21+08:00"
+++


# 什么是grpc

grpc官网：[https://www.grpc.io/](https://www.grpc.io/)

> A high-performance, open-source universal RPC framework

这个是官方对他的解释。这里就出现了一个新的名称RPC。什么是RPC呢?RPC(remote procedure call 远程过程调用),所谓RPC框架实际是提供了一套机制，使得应用程序之间可以进行通信，而且也遵从server/client模型。使用的时候客户端调用server端提供的接口就像是调用本地的函数一样。

RPC结构图：

![](https://gitee.com/coder2m/pic/raw/master/img/20200710082403.png)

RPC调用过程：

![](https://gitee.com/coder2m/pic/raw/master/img/20200710082506.png)

GRPC结构图：

![](https://gitee.com/coder2m/pic/raw/master/img/20200710082558.png)

性能对比可以查看这篇文章：[https://blog.csdn.net/xuduorui/article/details/77938644](https://blog.csdn.net/xuduorui/article/details/77938644)


# 准备工作

先安装Protobuf 编译器 protoc，下载地址：[https://github.com/protocolbuffers/protobuf/releases](https://github.com/protocolbuffers/protobuf/releases)

Protobuf插件库安装：
```golang
// gRPC运行时接口编解码支持库
go get -u github.com/golang/protobuf/proto
// 从 Proto文件(gRPC接口描述文件) 生成 go文件 的编译器插件
go get -u github.com/golang/protobuf/protoc-gen-go

```

go依赖包
```golang
go get -u google.golang.org/grpc

```

# 定义服务，编写proto文件

编写proto文件的语法什么的，我后面会单独出一篇文章。这里就不多说语法了。

```proto
syntax = "proto3";
package study;

//  请求参数
message Token {
    string jwtToken = 1; //1为字段顺序
    int64 timeStamp = 2;
}

//  返回参数
message TokenR {
    bool IsLogin = 1;
}

//  定义服务Auth
//  服务方法CheckToken
//  Token传入的值
//  TokenR输出的值
//  生成兼容grpc的go文件
//  protoc --go_out=plugins=grpc:. *.proto
service Auth {
    rpc CheckToken (Token) returns (TokenR);
}

```

# 生成兼容grpc的go文件
```golang
protoc --go_out=plugins=grpc:. *.proto

```
生成了对应的pb文件：
![](https://gitee.com/coder2m/pic/raw/master/img/20200710083457.png)

# 编写服务端
```golang
package main

import (
   "context"
   "fmt"
   study "goStudy/grpc/grpc001/auth"
   "google.golang.org/grpc"
   "net"
)

type AuthImp struct {
}

func (a *AuthImp) CheckToken(ctx context.Context, token *study.Token) (*study.TokenR, error) {
   s := token.JwtToken
   fmt.Println(s)
   /**
   一些列处理
   */
   //数据的返回
   res := new(study.TokenR)
   res.IsLogin = true
   return res, nil
}

func main() {
   server := grpc.NewServer()
   //注册
   study.RegisterAuthServer(server, new(AuthImp))

   lis, err := net.Listen("tcp", ":8091")
   if err != nil {
      panic(err.Error())
   }
   _ = server.Serve(lis)
}

```

# 编写客户端

```golang
package main

import (
   "context"
   "fmt"
   study "goStudy/grpc/grpc001/auth"
   "google.golang.org/grpc"
   "time"
)

func main() {
   //客户端连接
   conn, err := grpc.Dial("localhost:8091", grpc.WithInsecure())
   if err != nil {
      panic(err.Error())
   }
   defer conn.Close()

   authServiceClient := study.NewAuthClient(conn)

   request := study.Token{
      TimeStamp: time.Now().Unix(),
      JwtToken:  "3213213",
   }

   checkTokenClient, err := authServiceClient.CheckToken(context.Background(), &request)
   fmt.Println(checkTokenClient.IsLogin)
}

```

这样就完成了最简单的grpc功能。

这样的rpc接口只能在程序之间进行调用，但是我们最后的服务需要提供给用户使用，需要的是http请求。这里有几个方案进行解决：

  > 1.编写专门的提供http的web服务，web服务充当客户端，调用grpc。

  > 2.使用GRPC Gateway: [https://github.com/grpc-ecosystem/grpc-gateway/](https://github.com/grpc-ecosystem/grpc-gateway/)

  > 3.使用第三方拓展：[https://github.com/vaporz/turbo](https://github.com/vaporz/turbo)

