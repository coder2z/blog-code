+++
tags = ["golang", "go-micro", "微服务", "protobuf"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Protoc-go修改生成的文件的结构体的tag"
date = "2020-07-07T09:22:38+08:00"
+++


# 使用场景
在使用protoc生成的文件中生成的结构体的json没有小写，有时候我们就需要使用小写，最主要的就是在使用gin的时候还需要使用bind来绑定上传的东西。但是生成的*.pd.go又不建议我们去改。这时候就需要使用protoc-go-inject-tag，这里还有个更重要的地方，这里简单描述下：生成的结构体中的tag里面有个```omitempty```，这个的作用呢，就是在数据传输过程中，自动去掉```false 0 ""```这些数据。这里简单的举个例子，比如你的grpc服务端返回给客户端一个值为false的bool类型数据。但是在客户端接受的时候，这个字段就直接没有了。所以这样肯定是不行的。所以这里需要集体的去掉tag中的```omitempty```。这个时候protoc-go-inject-tag就可以满足这个需求。在之后的微服务（go-micro）开发中，这个也是很重要的地方！

![](https://gitee.com/coder2m/pic/raw/master/img/20200710092950.png)

git地址：[https://github.com/favadi/protoc-go-inject-tag](https://github.com/favadi/protoc-go-inject-tag)


# 安装

```golang
go get github.com/favadi/protoc-go-inject-tag

```

# 使用

## 只需要在.proto文件中写注释

```proto
syntax = "proto3";
package study;

//  请求参数
message Token {
    // @inject_tag: json:"jwtToken" form:"jwtToken"
    string jwtToken = 1; //1为字段顺序
    // @inject_tag: json:"timeStamp" form:"timeStamp"
    int64 timeStamp = 2;
}

//  返回参数
message TokenRequest {
    // @inject_tag: json:"isLogin" form:"isLogin"
    bool IsLogin = 1;
}

//  定义服务Auth
//  服务方法CheckToken
//  Token传入的值
//  TokenRequest输出的值
//  生成兼容grpc的go文件
//  protoc --go_out=plugins=grpc:. *.proto
service Auth {
    rpc CheckToken (Token) returns (TokenRequest);
}

```

## 执行生成pb文件

```golang
protoc --go_out=plugins=grpc:. *.proto

```

## 执行修改tag 命令

```golang
protoc-go-inject-tag -input=./*.pb.go

```

![](https://gitee.com/coder2m/pic/raw/master/img/20200710095030.png)

## 执行结果

这个时候查看生成的文件就变成了这样

```golang
//  请求参数
type Token struct {
	state         protoimpl.MessageState
	sizeCache     protoimpl.SizeCache
	unknownFields protoimpl.UnknownFields

	// @inject_tag: json:"jwtToken" form:"jwtToken"
	JwtToken string `protobuf:"bytes,1,opt,name=jwtToken,proto3" json:"jwtToken" form:"jwtToken"` //1为字段顺序
	// @inject_tag: json:"timeStamp" form:"timeStamp"
	TimeStamp int64 `protobuf:"varint,2,opt,name=timeStamp,proto3" json:"timeStamp" form:"timeStamp"`
}
//  返回参数
type TokenRequest struct {
	state         protoimpl.MessageState
	sizeCache     protoimpl.SizeCache
	unknownFields protoimpl.UnknownFields

	// @inject_tag: json:"isLogin" form:"isLogin"
	IsLogin bool `protobuf:"varint,1,opt,name=IsLogin,proto3" json:"isLogin" form:"isLogin"`
}

```

这样就行了，还可以根据自己的需求，自定义tag