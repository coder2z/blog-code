+++
tags = ["golang"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang-mod"
date = "2020-07-21T11:13:36+08:00"
+++


# Golang-mod

go modules 是相关Go包的集合。modules是源代码交换和版本控制的单元。 go命令直接支持使用modules，包括记录和解析对其他模块的依赖性。modules替换旧的基于GOPATH的方法来指定在给定构建中使用哪些源文件。(go官方的解释)

# 使用Mod

## 开启go mod
```go
go env -w GO111MODULE=on  // 开启go mod

```

## go mod有以下命令：
|  命令   | 说明  |
|  ----  | ----  |
| download  | download modules to local cache(下载依赖包) |
| edit  | edit go.mod from tools or scripts(编辑go.mod) |
| graph  | 	print module requirement graph (打印模块依赖图) |
| init  | initialize new module in current directory（在当前目录初始化mod） |
| tidy  | add missing and remove unused modules(拉取缺少的模块，移除不用的模块) |
| vendor  | make vendored copy of dependencies(将依赖复制到vendor下) |
| verify  | verify dependencies have expected content (验证依赖是否正确） |
| why  | explain why packages or modules are needed(解释为什么需要依赖) |

## 使用replace替换无法直接获取的package

由于某些已知的原因，并不是所有的package都能成功下载，比如：golang.org下的包。

modules 可以通过在 go.mod 文件中使用 replace 指令替换成github上对应的库，比如：

```
replace (
    golang.org/x/crypto v0.0.0-20190313024323-a1f597ede03a => github.com/golang/crypto v0.0.0-20190313024323-a1f597ede03a
)

```

**也可以replace使用导入自定义包**

在我们开发微服务的适合，每个微服务都需要一个mod，比如这样：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/21/20200721112455.png)

但是这样会出现问题，就是在不使用Gopath的情况下，不方便导入pb包，所有这是就需要自定义包：

只需要在需要使用pb包的mod中添加就行

```go
require (
	github.com/gin-gonic/gin v1.6.3
	github.com/micro/go-micro/v2 v2.9.1
	micro-auth/auth-srv v0.0.0
)

replace micro-auth/auth-srv => ../auth-srv

```

这样代码中就可以直接导入了

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/21/20200721115154.png)
