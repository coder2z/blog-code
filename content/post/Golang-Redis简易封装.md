+++
tags = ["golang"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang-Redis简易封装"
date = "2020-07-01T21:12:47+08:00"
+++


# Golang Redis 

## 下载依赖包

```golang
go get github.com/gomodule/redigo

```

## Redis 操作封装

### 配置文件

```ini
[redis]
Host = 127.0.0.1:6379
Password =
MaxIdle = 30
MaxActive = 30
IdleTimeout = 200

```

### golang代码
```golang
package Redis

import (
	"encoding/json"
	"time"
	"wPan/v1/Config" //加载配置

	"github.com/gomodule/redigo/redis"
)

//定义redis连接池
var Conn *redis.Pool

//初始化
func InitRedis() error {
	Conn = &redis.Pool{
		MaxIdle:     Config.RedisSetting.MaxIdle,
		MaxActive:   Config.RedisSetting.MaxActive,
		IdleTimeout: Config.RedisSetting.IdleTimeout,
		Dial: func() (redis.Conn, error) {
			c, err := redis.Dial("tcp", Config.RedisSetting.Host)
			if err != nil {
				return nil, err
			}
			if Config.RedisSetting.Password != "" {
				if _, err := c.Do("AUTH", Config.RedisSetting.Password); err != nil {
					_ = c.Close()
					return nil, err
				}
			}
			return c, err
		},
		TestOnBorrow: func(c redis.Conn, t time.Time) error {
			_, err := c.Do("PING")
			return err
		},
	}

	return nil
}

//redis 设置函数
func Set(key string, data interface{}, time int) (bool, error) {
	conn := Conn.Get()
	defer conn.Close()

	value, err := json.Marshal(data)
	if err != nil {
		return false, err
	}

	reply, err := redis.String(conn.Do("SET", key, value))
	_, _ = conn.Do("EXPIRE", key, time)
	return reply == "OK", err
}

//redis 检测存在函数
func Exists(key string) bool {
	conn := Conn.Get()
	defer conn.Close()

	exists, err := redis.Bool(conn.Do("EXISTS", key))
	if err != nil {
		return false
	}

	return exists
}

//redis 获取函数
func Get(key string) ([]byte, error) {
	conn := Conn.Get()
	defer conn.Close()

	reply, err := redis.Bytes(conn.Do("GET", key))
	if err != nil {
		return nil, err
	}

	return reply, nil
}

//redis 删除函数
func Delete(key string) (bool, error) {
	conn := Conn.Get()
	defer conn.Close()

	return redis.Bool(conn.Do("DEL", key))
}

//redis 模糊删除函数
func LikeDeletes(key string) error {
	conn := Conn.Get()
	defer conn.Close()

	keys, err := redis.Strings(conn.Do("KEYS", "*"+key+"*"))
	if err != nil {
		return err
	}

	for _, key := range keys {
		_, err = Delete(key)
		if err != nil {
			return err
		}
	}

	return nil
}

```

## 使用

### 初始化
```golang
if err := Redis.InitRedis(); err != nil {
	log.Println("init redis failed, err:" + err.Error())
	return
}

```
### 操作
```golang
...
if Redis.Exists("Register_" + s.Email) {
	return R.SENDCODE_EXISTS, false
}
...
if Redis.Get("Register_" + s.Email) {
	return R.SENDCODE_EXISTS, false
}
...

```
使用很简单这里就不一一举例了。通过这样的封装redis操作就变得更加简单了。