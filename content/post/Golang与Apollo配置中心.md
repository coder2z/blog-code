+++
tags = ["golang", "Apollo", "配置中心"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang与Apollo配置中心"
date = "2020-07-18T07:53:14+08:00"
+++


# Apollo - A reliable configuration management system

Apollo（阿波罗）是携程框架部门研发的分布式配置中心，能够集中化管理应用不同环境、不同集群的配置，配置修改后能够实时推送到应用端，并且具备规范的权限、流程治理等特性，适用于微服务配置管理场景。

服务端基于Spring Boot和Spring Cloud开发，打包后可以直接运行，不需要额外安装Tomcat等应用容器。

## 安装(Quick-Start)

Quick-Start 安装 是最简单的安装方案。不过这里需要注意的是，Quick-Start只针对本地测试使用，如果要部署到生产环境，还请另行参考分布式部署指南。

### 准备工作

#### Java

Java 1.8+

```sh
> java -version

java version "1.8.0_74"
Java(TM) SE Runtime Environment (build 1.8.0_74-b02)
Java HotSpot(TM) 64-Bit Server VM (build 25.74-b02, mixed mode)

```

#### MySQL

MySQL 5.6.5+

```sql
SHOW VARIABLES WHERE Variable_name = 'version';

```

#### 下载Quick-Start安装包

项目地址：[https://github.com/nobodyiam/apollo-build-scripts](https://github.com/nobodyiam/apollo-build-scripts)

```sh
git clone github.com/nobodyiam/apollo-build-scripts

```

## 安装步骤

### 创建数据库

Apollo服务端共需要两个数据库：ApolloPortalDB和ApolloConfigDB，我们把数据库、表的创建和样例数据都分别准备了sql文件，只需要导入数据库即可。

然后分别导入数据库文件：

```sh
source /项目地址/sql/apolloportaldb.sql

source /项目地址/sql/apolloconfigdb.sql

```

### 配置数据库连接信息

```sh
vim demo.sh

```

```yml
#apollo config db info
apollo_config_db_url=jdbc:mysql://localhost:3306/ApolloConfigDB?characterEncoding=utf8
apollo_config_db_username=用户名
apollo_config_db_password=密码（如果没有密码，留空即可）

# apollo portal db info
apollo_portal_db_url=jdbc:mysql://localhost:3306/ApolloPortalDB?characterEncoding=utf8
apollo_portal_db_username=用户名
apollo_portal_db_password=密码（如果没有密码，留空即可）

```

## 启动Apollo配置中心

### 确保端口未被占用

```sh
lsof -i:8080

```

### 执行启动脚本

```sh
./demo.sh start

```

当看到如下输出后，就说明启动成功了！

```log
==== starting service ====
Service logging file is ./service/apollo-service.log
Started [10768]
Waiting for config service startup.......
Config service started. You may visit http://localhost:8080 for service status now!
Waiting for admin service startup....
Admin service started
==== starting portal ====
Portal logging file is ./portal/apollo-portal.log
Started [10846]
Waiting for portal startup......
Portal started. You can visit http://localhost:8070 now!

```

浏览器打开：http://域名或者ip:8070/ 验证安装

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/18/20200718081558.png)

# Golang-Apollo客户端

## 安装依赖包

```sh
go get -u github.com/shima-park/agollo

```

## 使用

官方文档：[https://github.com/shima-park/agollo](https://github.com/shima-park/agollo)

### 获取配置

Apollo中的配置：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/18/20200718083852.png)

最简单的获取配置信息：
```golang
package main

import (
	"fmt"
	"github.com/shima-park/agollo"
)

func main() {
	a, err := agollo.New("Apollo搭建的ip或者域名:8080", "wpan", agollo.AutoFetchOnCacheMiss())
	if err != nil {
		panic(err)
	}

	fmt.Println(
		// 配置项foo的value
		//a.Get("rabbitmq_url"),

		// namespace为test.json的所有配置项
		//a.GetNameSpace("test.json"),

		// namespace为Service.Mysql的所有配置项
		a.GetNameSpace("Service.Mysql"),
		
		// 默认值, 为xxx提供一个默认bar
		//a.Get("xxx", agollo.WithDefault("bar")),

		// namespace为other_namespace, key为foo的value
		//a.Get("foo", agollo.WithNamespace("other_namespace")),
	)
}

```

运行结果:

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/18/20200718083940.png)

## 配置监听

```golang
a, err := agollo.New("localhost:8080", "your_appid", agollo.AutoFetchOnCacheMiss())
// error handle...

errorCh := a.Start()  // Start后会启动goroutine监听变化，并更新agollo对象内的配置cache
// 或者忽略错误处理直接 a.Start()

watchCh := a.Watch()

for{
	select{
	case err := <- errorCh:
		// handle error
	case resp := <-watchCh:
			fmt.Println(
			    "Namesapce:", resp.Namesapce,
			    "OldValue:", resp.OldValue,
			    "NewValue:", resp.NewValue,
			    "Error:", resp.Error,
			)
	}
}

```

还有很多用法可以参考官方文档。


参考连接：[https://github.com/ctripcorp/apollo](https://github.com/ctripcorp/apollo)

参考连接：[https://github.com/shima-park/agollo](https://github.com/shima-park/agollo)