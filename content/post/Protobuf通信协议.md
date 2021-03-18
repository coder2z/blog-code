+++
tags = ["protobuf", "微服务", "rpc"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Protobuf通信协议"
date = "2020-07-12T15:52:44+08:00"
+++


# RPC的调用过程

一个正常的RPC过程可以分为一下几个步骤：
 * client调用client stub，这是一次本地过程调用。
 * client stub将参数打包成一个消息，然后发送这个消息。打包过程也叫做marshalling。
 * client所在的系统将消息发送给server。
 * server的的系统将收到的包传给server stub。
 * server stub解包得到参数。 解包也被称作 unmarshalling。
 * server stub调用服务过程。返回结果按照相反的步骤传给client。

在上述的步骤实现远程接口调用时，所需要执行的函数是存在于远程机器中，即函数是在另外一个进程中执行的。因此，就带来了几个新问题：

* 1、Call ID映射。远端进程中间可以包含定义的多个函数，本地客户端该如何告知远端进程程序调用特定的某个函数呢？因此，在RPC调用过程中，所有的函数都需要有一个自己的ID。开发者在客户端（调用端）和服务端（被调用端）分别维护一个{函数<–>Call ID}的对应表。两者的表不一定完全相同，但是相同的函数对应的Call ID必须相同。当客户端需要进行远程调用时，调用者通过映射表查询想要调用的函数的名称，找到对应的Call ID，然后传递给服务端，服务端也通过查表，来确定客户端所需要调用的函数，然后执行相应函数的代码。
* 2、序列化与反序列化。客户端如何把参数传递给远程调用的函数呢？在本地调用中，我们只需要把参数压到栈里，然后让函数自己去栈里读就行。但是在远程过程调用时，客户端跟服务端是不同的进程，不能通过内存来传递参数。甚至有时候客户端和服务端使用的都不是同一种语言（比如服务端用C++，客户端用Java或者Python）。这时候就需要客户端把参数先转成一个字节流，传给服务端后，再把字节流转成自己能读取的格式。这个过程叫序列化和反序列化。同理，从服务端返回的值也需要序列化反序列化的过程。
* 3、网络传输。远程调用往往用在网络上，客户端和服务端是通过网络连接的。所有的数据都需要通过网络传输，因此就需要有一个网络传输层。网络传输层需要把Call ID和序列化后的参数字节流传递给服务端，然后在把序列化后的调用结果传回给客户端，完成这种数据传递功能的被成为传输层。大部分的网络传输成都使用TCP协议，属于长连接。

有对传递的数据进行序列化和反序列化的操作，这就是我们这里要说的：Protobuf。

# 简介

Google Protocol Buffer( 简称 Protobuf)是Google公司内部的混合语言数据标准，他们主要用于RPC系统和持续数据存储系统。

Protocol Buffers 是一种轻便高效的结构化数据存储格式，可以用于结构化数据串行化，或者说序列化。它很适合做数据存储或RPC数据交换格式。可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。简单来说，Protobuf的功能类似于XML，即负责把某种数据结构的信息，以某种格式保存起来。主要用于数据存储、传输协议等使用场景。


# 安装

## 安装protobuf编译器

可以在如下地址：[https://github.com/protocolbuffers/protobuf/releases](https://github.com/protocolbuffers/protobuf/releases)选择适合自己系统的Proto编译器程序进行下载并解压。然后配置环境变量：将protocke执行文件所在目录添加到当前系统的环境变量中。windows系统下可以直接在Path目录中进行添加。

## 安装go依赖

```golang
go get github.com/golang/protobuf/protoc-gen-go

```

# Protobuf 协议语法

## message：

Protobuf中定义一个数据结构需要用到关键字message，这一点和Java的class，Go语言中的struct类似。

## 标识号：

在消息的定义中，每个字段等号后面都有唯一的标识号，用于在反序列化过程中识别各个字段的，一旦开始使用就不能改变。标识号从整数1开始，依次递增，每次增加1，标识号的范围为1~2^29 – 1，其中[19000-19999]为Protobuf协议预留字段，开发者不建议使用该范围的标识号；一旦使用，在编译时Protoc编译器会报出警告。


## 字段规则：
字段规则有三种：
* 1.required：该规则规定，消息体中该字段的值是必须要设置的。
* 2.optional：消息体中该规则的字段的值可以存在，也可以为空，optional的字段可以根据defalut设置默认值。
* 3.repeated：消息体中该规则字段可以存在多个（包括0个），该规则对应java的数组或者go语言的slice。

## 数据类型：
常见的数据类型与protoc协议中的数据类型映射如下：
![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/12/20200712161553.png)

## 枚举类型：

proto协议支持使用枚举类型

```golang
enum Sex{
    male=1;
    female=2;
}

```

## 字段默认值：

```golang
message Address {
    required sint32 id = 1 [default = 1];
    required string name = 2 [default = '北京'];
    optional string pinyin = 3 [default = 'beijing'];
    required string address = 4;
    required bool flag = 5 [default = true];
}

```

## 导入

如果需要引用的message是写在别的.proto文件中，可以通过import "xxx.proto"来进行引入

## 嵌套

```golang
syntax = "proto3";
package example;
message Person {
    required string Name = 1;
    required int32 Age = 2;
    required string From = 3;
    optional Address Addr = 4;
    message Address {
        required sint32 id = 1;
        required string name = 2;
        optional string pinyin = 3;
        required string address = 4;
    }
}

```
# 生成文件
这里看你，需要使用什么语言的，如果你需要再c和py之间传输数据，你就需要生成对应的c和py文件，就行。这里我因为使用的golang。我就生成go的

```golang
protoc --go_out=. *.proto

```

## 这里推荐下几个插件（golang）

* protoc-gen-gogo：和protoc-gen-go生成的文件差不多，性能也几乎一样(稍微快一点点)

* protoc-gen-gofast：生成的文件更复杂，性能也更高(快5-7倍)

```golang
//gogo
go get github.com/gogo/protobuf/protoc-gen-gogo

//gofast
go get github.com/gogo/protobuf/protoc-gen-gofast

```

安装gogoprotobuf库文件

```golang
go get github.com/gogo/protobuf/proto
go get github.com/gogo/protobuf/gogoproto  //这个不装也没关系

```

生成文件

```golang
//gogo
protoc --gogo_out=. *.proto

//gofast
protoc --gofast_out=. *.proto

```

性能测试

```golang
//goprotobuf
"编码"：447ns/op
"解码"：422ns/op

//gogoprotobuf-go
"编码"：433ns/op
"解码"：427ns/op

//gogoprotobuf-fast
"编码"：112ns/op
"解码"：112ns/op

```
# 使用

## 发送消息

把消息进行编码

```golang

book := &pb.AddressBook{}
// ...

out, err := proto.Marshal(book)
if err != nil {
        log.Fatalln("Failed to encode address book:", err)
}
if err := ioutil.WriteFile(fname, out, 0644); err != nil {
        log.Fatalln("Failed to write address book:", err)
}

```

## 接收消息

把消息进行解码

```golang
in, err := ioutil.ReadFile(fname)
if err != nil {
        log.Fatalln("Error reading file:", err)
}
book := &pb.AddressBook{}
if err := proto.Unmarshal(in, book); err != nil {
        log.Fatalln("Failed to parse address book:", err)
}

```

参考连接：[https://developers.google.com/protocol-buffers/docs/gotutorial](https://developers.google.com/protocol-buffers/docs/gotutorial)