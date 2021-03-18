+++
tags = ["golang", "RabbitMQ", "消息队列"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang实现RabbitMQ五种模式"
date = "2020-07-03T09:16:26+08:00"
+++

# 使用的依赖包
```golang
github.com/streadway/amqp

```

# 创建RabbitMQ实例

```golang
package RabbitMQ

import (
	"fmt"
	"github.com/streadway/amqp"
	"log"
)

// 用户名 密码 ip:端口/虚拟机
const MQURL = "amqp://admin:123456@127.0.0.1:5672/test"

type RabbitMQ struct {
	conn    *amqp.Connection
	channel *amqp.Channel
	//队列名称
	QueueName string
	//交换机
	Exchange string
	//key
	key string
	//连接信息
	MqUrl string
}

//创建RabbitMQ结构体实例
func NewRabbitMQ(queueName, exchange, key string) *RabbitMQ {
	rabbitmq := &RabbitMQ{QueueName: queueName, Exchange: exchange, key: key, MqUrl: MQURL}
	var err error
	//创建RabbitMQ连接
	rabbitmq.conn, err = amqp.Dial(rabbitmq.MqUrl)
	rabbitmq.failOnErr(err, "创建连接错误!")
	rabbitmq.channel, err = rabbitmq.conn.Channel()
	rabbitmq.failOnErr(err, "获取channel失败!")
	return rabbitmq
}

//断开channel和connection
func (r *RabbitMQ) Destroy() {
	r.channel.Close()
	r.conn.Close()
}

//错误处理函数
func (r *RabbitMQ) failOnErr(err error, message string) {
	if err != nil {
		log.Fatalf("%s:%s", message, err)
		panic(fmt.Sprintf("%s,%s", message, err))
	}
}

```

# Simple模式

Simple模式工作流程：

![Simple模式](https://gitee.com/coder2m/pic/raw/master/img/20200708092914.png)

```golang
//创建简单模式下RabbitMQ实例
func NewRabbitMQSimple(queueName string) *RabbitMQ {
	return NewRabbitMQ(queueName, "", "")
}

//简单模式下生产代码
func (r *RabbitMQ) PublishSimple(message string) {
	//1.申请队列,如果队列不存在会自动创建,如果存在则跳过创建
	//保证队列存在,消息队列能发送到队列中
	_, err := r.channel.QueueDeclare(
		r.QueueName,
		//是否持久化
		false,
		//是否为自动删除
		false,
		//是否具有排他性
		false,
		//是否阻塞
		false,
		//额外属性
		nil,
	)
	if err != nil {
		fmt.Println("QueueDeclare:", err)
	}

	//2.发送消息到队列中
	err = r.channel.Publish(
		r.Exchange,
		r.QueueName,
		//如果为true,根据exchange类型和routekey规则,如果无法找到符合条件的队列那么会把发送的消息返回给发送者
		false,
		//如果为true,当exchange发送消息队列到队列后发现队列上没有绑定消费者,则会把消息发还给发送者
		false,
		amqp.Publishing{
			ContentType: "text/plain",
			Body:        []byte(message),
		})
	if err != nil {
		fmt.Println("Publish:", err)
	}
}

//简单模式下消费代码
func (r *RabbitMQ) ConsumeSimple() {
	//1.申请队列,如果队列不存在会自动创建,如果存在则跳过创建
	//保证队列存在,消息队列能发送到队列中
	_, err := r.channel.QueueDeclare(
		r.QueueName,
		//是否持久化
		false,
		//是否为自动删除
		false,
		//是否具有排他性
		false,
		//是否阻塞
		false,
		//额外属性
		nil)
	if err != nil {
		fmt.Println("QueueDeclare:", err)
	}

	//2.接受消息
	msgs, err := r.channel.Consume(
		r.QueueName,
		//用来区分多个消费者
		"",
		//是否自动应答
		true,
		//是否具有排他性
		false,
		//如果设置为true,表示不能将同一个connection中发送消息传递给这个connection中的消费者
		false,
		//队列消费是否阻塞
		false,
		nil)
	if err != nil {
		fmt.Println("Consume:", err)
	}

	forever := make(chan bool)
	//3.启用协程处理消息
	go func() {
		for d := range msgs {
			//实现我们要处理的逻辑函数
			log.Printf("Received a message:%s", d.Body)
		}
	}()
	log.Printf("[*] Waiting for messages,To exit press CTRL+C\n")
	<-forever
}

```

## 简单模式publish

```golang
package main

import (
	"fmt"
	"go-rabbitmq/RabbitMQ"
)

func main()  {
	rabbitmq := RabbitMQ.NewRabbitMQSimple("test")
	rabbitmq.PublishSimple("myxy99.cn msg")
	fmt.Println("发送成功!")
}

```

## 简单模式recevie

```golang
package main

import "rabbitmq/RabbitMQ"

func main() {
	rabbitmq := RabbitMQ.NewRabbitMQSimple("test")
	rabbitmq.ConsumeSimple()
}

```

# Work模式

Work模式工作流程：

![Work模式](https://gitee.com/coder2m/pic/raw/master/img/20200708093050.png)

simple模式和work模式其实用的是一套逻辑代码，只是work模式是可以有多个消费者的，work模式起到一个负载均衡的作用。

## work模式publish

```golang
package main

import (
	"fmt"
	"rabbitmq/RabbitMQ"
	"strconv"
	"time"
)

func main() {
	rabbitmq := RabbitMQ.NewRabbitMQSimple("test")

	for i := 0; i <= 100; i++ {
		rabbitmq.PublishSimple("Hello test!" + strconv.Itoa(i))
		time.Sleep(1 * time.Second)
		fmt.Println(i)
	}
}

```

## work模式receive1

```golang
package main

import "rabbitmq/RabbitMQ"

func main() {
	rabbitmq := RabbitMQ.NewRabbitMQSimple("test")
	rabbitmq.ConsumeSimple()
}

```

## work模式receive2

```golang
package main

import "rabbitmq/RabbitMQ"

func main() {
	rabbitmq := RabbitMQ.NewRabbitMQSimple("test")
	rabbitmq.ConsumeSimple()
}

```

# 订阅模式

订阅模式工作流程：

![](https://gitee.com/coder2m/pic/raw/master/img/20200708095831.png)

订阅模式的特别是：一个消息被投递到多个队列，一个消息能被多个消费者获取。过程是由生产者将消息发送到exchange(交换机）里，然后exchange通过一系列的规则发送到队列上，然后由绑定对应的消费者进行消息。

```golang
//订阅模式创建RabbitMQ实例
func NewRabbitMQPubSub(exchangeName string) *RabbitMQ {
	//创建RabbitMQ实例
	rabbitmq := NewRabbitMQ("",exchangeName,"")
	var err error
	//获取connection
	rabbitmq.conn, err = amqp.Dial(rabbitmq.Mqurl)
	rabbitmq.failOnErr(err,"failed to connect rabbitmq!")
	//获取channel
	rabbitmq.channel, err = rabbitmq.conn.Channel()
	rabbitmq.failOnErr(err, "failed to open a channel")
	return rabbitmq
}

//订阅模式生产
func (r *RabbitMQ) PublishPub(message string) {
	//1.尝试创建交换机
	err := r.channel.ExchangeDeclare(
		r.Exchange,
		"fanout",
		true,
		false,
		//true表示这个exchange不可以被client用来推送消息，仅用来进行exchange和exchange之间的绑定
		false,
		false,
		nil,
	)

	r.failOnErr(err, "Failed to declare an excha"+
		"nge")

	//2.发送消息
	err = r.channel.Publish(
		r.Exchange,
		"",
		false,
		false,
		amqp.Publishing{
			ContentType: "text/plain",
			Body:        []byte(message),
		})
}

//订阅模式消费端代码
func (r *RabbitMQ) RecieveSub() {
	//1.试探性创建交换机
	err := r.channel.ExchangeDeclare(
		r.Exchange,
		//交换机类型
		"fanout",
		true,
		false,
		//YES表示这个exchange不可以被client用来推送消息，仅用来进行exchange和exchange之间的绑定
		false,
		false,
		nil,
	)
	r.failOnErr(err, "Failed to declare an exch"+
		"ange")
    //2.试探性创建队列，这里注意队列名称不要写
	q, err := r.channel.QueueDeclare(
		"", //随机生产队列名称
		false,
		false,
		true,
		false,
		nil,
	)
	r.failOnErr(err, "Failed to declare a queue")

	//绑定队列到 exchange 中
	err = r.channel.QueueBind(
		q.Name,
		//在pub/sub模式下，这里的key要为空
		"",
		r.Exchange,
		false,
		nil)

	//消费消息
	messges, err := r.channel.Consume(
		q.Name,
		"",
		true,
		false,
		false,
		false,
		nil,
	)

	forever := make(chan bool)

	go func() {
		for d := range messges {
			log.Printf("Received a message: %s", d.Body)
		}
	}()

	fmt.Println("退出请按 CTRL+C\n")
	<-forever
}

```

## 订阅模式publish

```golang
package main

import (
	"rabbitmq/RabbitMQ"
	"strconv"
	"time"
	"fmt"
)

func main() {
	rabbitmq := RabbitMQ.NewRabbitMQPubSub("newProduct")
	for i := 0; i < 100; i++ {
		rabbitmq.PublishPub("订阅模式生产第" +
			strconv.Itoa(i) + "条" + "数据")
		fmt.Println("订阅模式生产第" +
			strconv.Itoa(i) + "条" + "数据")
		time.Sleep(1 * time.Second)
	}

}

```

## 订阅模式receive1

```golang
package main

import "rabbitmq/RabbitMQ"

func main() {
	rabbitmq := RabbitMQ.NewRabbitMQPubSub("newProduct")
	rabbitmq.RecieveSub()
}

```

## 订阅模式receive2

```golang
package main

import "rabbitmq/RabbitMQ"

func main() {
	rabbitmq := RabbitMQ.NewRabbitMQPubSub("newProduct")
	rabbitmq.RecieveSub()
}

```

# 路由模式

路由模式工作流程：

![](https://gitee.com/coder2m/pic/raw/master/img/20200708100024.png)

路由模式:一个消息由多个消费者消费的基础上指定由哪些消息者来消费。

```golang
func NewRabbitMQRouting(exchangeName string,routingKey string) *RabbitMQ {
	//创建RabbitMQ实例
	rabbitmq := NewRabbitMQ("",exchangeName,routingKey)
	var err error
	//获取connection
	rabbitmq.conn, err = amqp.Dial(rabbitmq.Mqurl)
	rabbitmq.failOnErr(err,"failed to connect rabbitmq!")
	//获取channel
	rabbitmq.channel, err = rabbitmq.conn.Channel()
	rabbitmq.failOnErr(err, "failed to open a channel")
	return rabbitmq
}

//路由模式发送消息
func (r *RabbitMQ) PublishRouting(message string )  {
	//1.尝试创建交换机
	err := r.channel.ExchangeDeclare(
		r.Exchange,
		//要改成direct
		"direct",
		true,
		false,
		false,
		false,
		nil,
	)

	r.failOnErr(err, "Failed to declare an exchange")

	//2.发送消息
	err = r.channel.Publish(
		r.Exchange,
		//要设置
		r.Key,
		false,
		false,
		amqp.Publishing{
			ContentType: "text/plain",
			Body:        []byte(message),
		})
}
//路由模式接受消息
func (r *RabbitMQ) RecieveRouting() {
	//1.试探性创建交换机
	err := r.channel.ExchangeDeclare(
		r.Exchange,
		//交换机类型
		"direct",
		true,
		false,
		false,
		false,
		nil,
	)
	r.failOnErr(err, "Failed to declare an exch"+
		"ange")
	//2.试探性创建队列，这里注意队列名称不要写
	q, err := r.channel.QueueDeclare(
		"", //随机生产队列名称
		false,
		false,
		true,
		false,
		nil,
	)
	r.failOnErr(err, "Failed to declare a queue")

	//绑定队列到 exchange 中
	err = r.channel.QueueBind(
		q.Name,
		//需要绑定key
		r.Key,
		r.Exchange,
		false,
		nil)

	//消费消息
	messges, err := r.channel.Consume(
		q.Name,
		"",
		true,
		false,
		false,
		false,
		nil,
	)

	forever := make(chan bool)

	go func() {
		for d := range messges {
			log.Printf("Received a message: %s", d.Body)
		}
	}()

	fmt.Println("退出请按 CTRL+C\n")
	<-forever
}

```

## 路由模式publish

```golang
package main

import (
	"rabbitmq/RabbitMQ"
	"strconv"
	"time"
	"fmt"
)

func main()  {
	mqOne:=RabbitMQ.NewRabbitMQRouting("myxy99","myxy99_one")
	mqTwo:=RabbitMQ.NewRabbitMQRouting("myxy99","myxy99_two")
	for i := 0; i <= 10; i++ {
		mqOne.PublishRouting("Hello myxy99 one!" + strconv.Itoa(i))
		mqTwo.PublishRouting("Hello myxy99 Two!" + strconv.Itoa(i))
		time.Sleep(1 * time.Second)
		fmt.Println(i)
	}
}

```

## 路由模式receive-one

```golang
package main

import "rabbitmq/RabbitMQ"

func main()  {
	mq:=RabbitMQ.NewRabbitMQRouting("myxy99","myxy99_one")
	mq.RecieveRouting()
}

```

## 路由模式receive-two

```golang
package main

import "rabbitmq/RabbitMQ"

func main()  {
	mq:=RabbitMQ.NewRabbitMQRouting("myxy99","myxy99_two")
	mq.RecieveRouting()
}

```

# 话题模式

话题模式工作流程：

![](https://gitee.com/coder2m/pic/raw/master/img/20200708100143.png)

话题模式：话题模式是在路由模式上演化而来。不同的是我们以通配符的方式来指定我们的消费者。

```golang
//话题模式
//创建RabbitMQ实例
func NewRabbitMQTopic(exchangeName string,routingKey string) *RabbitMQ {
	//创建RabbitMQ实例
	rabbitmq := NewRabbitMQ("",exchangeName,routingKey)
	var err error
	//获取connection
	rabbitmq.conn, err = amqp.Dial(rabbitmq.Mqurl)
	rabbitmq.failOnErr(err,"failed to connect rabbitmq!")
	//获取channel
	rabbitmq.channel, err = rabbitmq.conn.Channel()
	rabbitmq.failOnErr(err, "failed to open a channel")
	return rabbitmq
}
//话题模式发送消息
func (r *RabbitMQ) PublishTopic(message string )  {
	//1.尝试创建交换机
	err := r.channel.ExchangeDeclare(
		r.Exchange,
		//要改成topic
		"topic",
		true,
		false,
		false,
		false,
		nil,
	)

	r.failOnErr(err, "Failed to declare an excha"+
		"nge")

	//2.发送消息
	err = r.channel.Publish(
		r.Exchange,
		//要设置
		r.Key,
		false,
		false,
		amqp.Publishing{
			ContentType: "text/plain",
			Body:        []byte(message),
		})
}
//话题模式接受消息
//要注意key,规则
//其中“*”用于匹配一个单词，“#”用于匹配多个单词（可以是零个）
//匹配 myxy99.* 表示匹配 myxy99.hello, 但是myxy99.hello.one需要用myxy99.#才能匹配到
func (r *RabbitMQ) RecieveTopic() {
	//1.试探性创建交换机
	err := r.channel.ExchangeDeclare(
		r.Exchange,
		//交换机类型
		"topic",
		true,
		false,
		false,
		false,
		nil,
	)
	r.failOnErr(err, "Failed to declare an exch"+
		"ange")
	//2.试探性创建队列，这里注意队列名称不要写
	q, err := r.channel.QueueDeclare(
		"", //随机生产队列名称
		false,
		false,
		true,
		false,
		nil,
	)
	r.failOnErr(err, "Failed to declare a queue")

	//绑定队列到 exchange 中
	err = r.channel.QueueBind(
		q.Name,
		//在pub/sub模式下，这里的key要为空
		r.Key,
		r.Exchange,
		false,
		nil)

	//消费消息
	messges, err := r.channel.Consume(
		q.Name,
		"",
		true,
		false,
		false,
		false,
		nil,
	)

	forever := make(chan bool)

	go func() {
		for d := range messges {
			log.Printf("Received a message: %s", d.Body)
		}
	}()

	fmt.Println("退出请按 CTRL+C\n")
	<-forever
}

```

## 话题模式publish

```golang
package main

import (
	"rabbitmq/RabbitMQ"
	"strconv"
	"time"
	"fmt"
)

func main()  {
	mqOne:=RabbitMQ.NewRabbitMQTopic("myxy99Topic","myxy99.topic.one")
	mqTwo:=RabbitMQ.NewRabbitMQTopic("myxy99Topic","myxy99.topic.two")
	for i := 0; i <= 10; i++ {
		mqOne.PublishTopic("Hello myxy99 topic one!" + strconv.Itoa(i))
		mqTwo.PublishTopic("Hello myxy99 topic Two!" + strconv.Itoa(i))
		time.Sleep(1 * time.Second)
		fmt.Println(i)
	}
	
}

```

## 话题模式receive-all

```golang
package main

import "rabbitmq/RabbitMQ"

func main()  {
	mq:=RabbitMQ.NewRabbitMQTopic("myxy99Topic","#")
	mq.RecieveTopic()
}

```

## 话题模式receive-two

```golang
package main

import "rabbitmq/RabbitMQ"

func main()  {
	mq:=RabbitMQ.NewRabbitMQTopic("myxy99Topic","myxy99.*.two")
	mq.RecieveTopic()
}

```

GitHub地址：[https://github.com/coder2m/Go-RabbitMQ](https://github.com/coder2m/Go-RabbitMQ)