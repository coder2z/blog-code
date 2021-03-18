+++
tags = ["golang", "爬虫"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang实现将Hexo博客文章推送到微信公众号"
date = "2020-07-15T10:53:59+08:00"
+++


# Golang实现将Hexo博客文章推送到微信公众号

最近在写博客的时候，就在想，能不能实现博客更新了然后就自动给别人提醒呢？比如每天提醒更新了什么博客。然后就有了这个项目，我的想法就是通过golang进行爬虫，把所有的文章都存储起来，获取到新的文章，然后把新的文章连接进行整合，发送到微信公众号，这样所有的关注了微信公众号的，就能收到，每天更新的新的博客。

# 实现爬虫

这里我选择的是colly爬虫框架。然后把爬取的数据进行数据库存储。
```golang
package main

import (
	"fmt"
	"github.com/gocolly/colly"
	"regexp"
	"strings"
	"wx-blog/config"
	Redis "wx-blog/redis"
	"wx-blog/utils"
)

func main() {

	i := config.Config{}
	conf := i.GetConf()
	redis := Redis.NewRedis(conf)

	//爬虫采集 收集到redis

	//获取需要爬取的地址
	webList := redis.HGetAll("BlogUrl").Val()
	for key, value := range webList {
		c := colly.NewCollector(
			colly.AllowedDomains(strings.Split(key, "//")[1]),
		)
		c.OnRequest(func(request *colly.Request) {
			//去重
			ok, _ := redis.HExists("BlogUrl", utils.GetMd5(request.URL.String())).Result()
			if ok {
				request.Abort()
				return
			}
			fmt.Println(request.URL.String())
		})
		c.OnHTML("a[href]", func(e *colly.HTMLElement) {
			//获取所有a
			_ = e.Request.Visit(e.Attr("href"))

		})
		c.OnHTML("title", func(e *colly.HTMLElement) {

			//文章页面正则提取
			matched, _ := regexp.MatchString(value, e.Request.URL.String())
			if matched {

				//最新的连接存储到redis
				redis.HSet("BlogUrl_req", e.Text, e.Request.URL.String())

				//去记录重库
				redis.HSet("BlogUrl_db", utils.GetMd5(e.Request.URL.String()), e.Text)

			}
		})
		_ = c.Visit(key)
	}
}

```

*爬虫前需要把需要爬虫的网站信息存储到redis中*

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/15/20200715123711.png)

爬取结果：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/15/20200715123621.png)


# 实现微信公众号数据的发送

微信发送使用的群发接口，需要服务号认证才行。这里使用的是测试账号。

weix.go

```golang
package weix

import (
	"encoding/json"
	"errors"
	"fmt"
	"wx-blog/request"
)

const (
	WxGetAccessToken = "https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=%v&secret=%v"
	WxSendText       = "https://api.weixin.qq.com/cgi-bin/message/mass/sendall?access_token=%v"
)

type WeiX struct {
	AppID       string
	AppSecret   string
	AccessToken string
}
//获取token方法
func (w *WeiX) GetToken() (err error) {
	url := fmt.Sprintf(WxGetAccessToken, w.AppID, w.AppSecret)
	i := &request.Request{}
	i.Call("GET", url, nil)
	if i.Err == nil {
		var data map[string]interface{}
		err = json.Unmarshal(i.Body, &data)
		if err == nil {
			a := data["access_token"]
			if a != nil {
				w.AccessToken = data["access_token"].(string)
				return
			}
			err = errors.New("请检查AppID，AppSecret")
			return
		}
	}
	return
}
//发送信息方法
func (w *WeiX) Send(message string) (err error) {
	url := fmt.Sprintf(WxSendText, w.AccessToken)
	i := &request.Request{}
	i.Call("POST", url, []byte(message))
	if i.Err == nil {
		var data map[string]interface{}
		err = json.Unmarshal(i.Body, &data)
		if err == nil {
			fmt.Println(data)
		}
	}
	return
}

```

wx.go

```golang
package main

import (
	"log"
	"wx-blog/config"
	Redis "wx-blog/redis"
	"wx-blog/weix"
)

func main() {
	i := config.Config{}
	conf := i.GetConf()
	redis := Redis.NewRedis(conf)

	//获取最新的文章：
	data, err := redis.HGetAll("BlogUrl_req").Result()

	if err != nil {
		log.Panic(err.Error())
	}

	if data == nil {
		log.Panic("没有新文章")
		return
	}

	//封装发送的数据
	var message string
	for key, value := range data {
		message += key + ":" + value + "<br>"
	}

	//初始化微信
	wx := weix.WeiX{
		AppSecret: conf.WX.AppSecret,
		AppID:     conf.WX.AppID,
	}
	//获取Token
	err = wx.GetToken()
	if err != nil {
		log.Panic(err.Error())
	}

	//封装消息
	sendMessage := `{
   "filter":{
      "is_to_all":false,
      "tag_id":100
   },
   "text":{
      "content":` + message + `
   },
    "msgtype":"text"
}`

	//发送消息
	err = wx.Send(sendMessage)
	if err != nil {
		log.Panic(err.Error())
	}

	//待发数据
	redis.HDel("BlogUrl_req")

	log.Println("文章发送成功!")
}

```

参考连接:[https://github.com/gocolly/colly](https://github.com/gocolly/colly)

源码地址:[https://github.com/coder2m/weixin-blog](https://github.com/coder2m/weixin-blog)

