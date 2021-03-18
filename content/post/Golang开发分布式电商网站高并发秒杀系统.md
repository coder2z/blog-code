+++
tags = ["golang", "分布式", "RabbitMQ"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang开发分布式电商网站高并发秒杀系统"
date = "2020-07-11T10:21:53+08:00"
+++


> 这个系统的主要目的在于秒杀，所有其他地方都做的很简单。基础功能不多！

# 技术栈：

> web框架：gin

> 消息队列：RabbitMQ

> 分布式方案：hash环

> orm: gorm

> 限流器：tollbooth

> 登录验证：jwt

# 架构:

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/12/20200712131203.png)

# 启动
cd /cmd

go run main.go //启动后台管理接口

go run client.go //启动RabbitMQ写入数据库客户端

go run spike.go //启动秒杀系统，支持横行扩展

# 测试

测试没有使用集群，只是一个服务器

这里我使用测试工具是jmeter


## 设置：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/12/20200712090309.png)


## 测试结果：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/12/20200712085943.png)


## RabbitMQ：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/12/20200712090446.png)

## mysql:
并没有超卖，测试添加了1000个库存

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/12/20200712090604.png)

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/12/20200712090629.png)


# 实现

这里主要说下最重要的秒杀流程。主要的核心逻辑在```spike_service.go```中。

项目在启动的时候，我们查询一下数据库，对数据库中需要秒杀的商品的信息，缓存到程序中，再后面的秒杀请求中，对其缓存进行修改，这样就实现了去redis实现。

具体的代码：

```golang
	//记录现在的秒杀商品的数量
	commodityCache map[int]*models.Commodity
	models.Init()
	models.MysqlHandler.AutoMigrate(models.Order{})
	repository := &repositories.CommodityRepository{Db: models.MysqlHandler}
	service := &services.CommodityService{CommodityRepository: repository}
    commodityList, err := service.GetCommodityAll()
  	for _, value := range *commodityList {
		commodityCache[int(value.ID)] = &value
  }
  
```

这样缓存到了程序中，后面的判断都再这个程序缓存中，速度就快很多。但是这也有个缺点，当你的秒杀商品很多的时候，可能导致你的程序内存使用很高。这里要注意的是，修改缓存数据的时候，记得加锁。

然后后面的具体操作就是通过一致性hash算法，进行选择服务器，如果计算出来的就是本机就本机进行操作，如果是其他ip就本机使用代理访问，这样就实现了分布式操作。

设计的东西还是比较多。直接看看源代码，应该还是能看懂。我项目的写的还是比较清楚的。感兴趣的可以看看。

项目github地址：[https://github.com/coder2m/shopping](https://github.com/coder2m/shopping)
