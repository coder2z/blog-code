+++
tags = ["golang"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang邮件发送"
date = "2020-06-27T19:27:48+08:00"
+++


# golang发送邮件

## 安装库

```golang
go get github.com/jordan-wright/email

```

## 简单代码实现

```golang
package main

import (
    "log"
    "net/smtp"
    "github.com/jordan-wright/email"
)

func main() {
    e := email.NewEmail()
    //设置发送方的邮箱
    e.From = "112233 <XXX@qq.com>"
    // 设置接收方的邮箱
    e.To = []string{"XXX@qq.com"}
    //设置主题
    e.Subject = "这是主题"
    //设置文件发送的内容
    e.Text = []byte("发送的内容")
    //设置服务器相关的配置
    err := e.Send("smtp.qq.com:25", smtp.PlainAuth("", "你的邮箱账号", "这块是你的授权码", "smtp.qq.com"))
    if err != nil {
       ....
    }
}

```

## 实现抄送功能

```golang
func main() {
    ...
    //设置抄送如果抄送多人逗号隔开
    e.Cc = []string{"XXX@qq.com",XXX@qq.com}
    //设置秘密抄送
    e.Bcc = []string{"XXX@qq.com"}
    ...
}

```
## 发送html代码的邮件

```golang
func main() {
    ...
    e.HTML = []byte(`
    <h1><a href="http://www.baidu.com/">百度</a></h1>    
    `)
    ...
}

```

## 实现邮件附件的发送

```golang
func main() {
    ...
    e.AttachFile("./test.txt")
    ...
}

```

## 连接池

每次调用Send时都会和 SMTP 服务器建立一次连接，如果发送邮件很多很频繁的话可能会有性能问题。email提供了连接池，可以复用网络连接：

```golang
package main

import (
    "fmt"
    "log"
    "net/smtp"
    "os"
    "sync"
    "time"
    "github.com/jordan-wright/email"
)

func main() {
    ch := make(chan *email.Email, 10)
    p, err := email.NewPool(
        "smtp.qq.com:25",
        4,
        smtp.PlainAuth("", "XXX@qq.com", "你的授权码", "smtp.qq.com"),
    )

    if err != nil {
        log.Fatal("failed to create pool:", err)
    }

    var wg sync.WaitGroup
    wg.Add(4)
    for i := 0; i < 4; i++ {
        go func() {
            defer wg.Done()
            for e := range ch {
                err := p.Send(e, 10*time.Second)
                if err != nil {
                    fmt.Fprintf(os.Stderr, "email:%v sent error:%v\n", e, err)
                }
            }
        }()
    }

    for i := 0; i < 10; i++ {
        e := email.NewEmail()
        e.From = "dj <XXX@qq.com>"
        e.To = []string{"XXX@qq.com"}
        e.Subject = "Awesome web"
        e.Text = []byte(fmt.Sprintf("Awesome Web %d", i+1))
        ch <- e
    }
    close(ch)
    wg.Wait()
}

```
上面程序中，我们创建 4 goroutine 共用一个连接池发送邮件，发送 10 封邮件后程序退出。为了等邮件都发送完成或失败，程序才退出，我们使用了sync.WaitGroup。由于使用了 goroutine，邮件顺序不能保证。

参考连接：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)