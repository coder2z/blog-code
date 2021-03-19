+++
tags = ["golang", "MongoDb"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang-MongoDb方法封装"
date = "2020-07-30T14:09:50+08:00"
+++



# Golang操作MongoDb

## 安装依赖

```go
go get -u github.com/globalsign/mgo
```

## MongoDb的方法封装

```go
/**
* @Author: myxy99 <myxy99@foxmail.com>
* @Date: 2020/7/30 14:16
 */
package mongoDb

import (
	"fmt"
	"github.com/globalsign/mgo"
	"log"
)

type ClientImp interface {
	connect(db, collection string) (*mgo.Session, *mgo.Collection)
	Insert(db, collection string, docs ...interface{}) error
	FindOne(db, collection string, query, selector, result interface{}) error
	FindAll(db, collection string, query, selector, result interface{}) error
	Update(db, collection string, query, update interface{}) error
	Remove(db, collection string, query interface{}) error
}

type Client struct {
	cfg CallerCfg
	m   *mgo.Session
}

type cLogger struct{}

func (c *cLogger) Output(calldepth int, s string) error {
	fmt.Println("calldepth: ", calldepth, ", s: ", s)
	return nil
}

func (c *Client) connect(db, collection string) (*mgo.Session, *mgo.Collection) {
	s := c.m.Copy()
	return s, s.DB(db).C(collection)
}

func (c *Client) Insert(db, collection string, docs ...interface{}) error {
	ms, mc := c.connect(db, collection)
	defer ms.Close()
	return mc.Insert(docs...)
}

func (c *Client) FindOne(db, collection string, query, selector, result interface{}) error {
	ms, mc := c.connect(db, collection)
	defer ms.Close()
	return mc.Find(query).Select(selector).One(result)
}

func (c *Client) FindAll(db, collection string, query, selector, result interface{}) error {
	ms, mc := c.connect(db, collection)
	defer ms.Close()
	return mc.Find(query).Select(selector).All(result)
}

func (c *Client) Update(db, collection string, query, update interface{}) error {
	ms, mc := c.connect(db, collection)
	defer ms.Close()
	return mc.Update(query, update)
}

func (c *Client) Remove(db, collection string, query interface{}) error {
	ms, mc := c.connect(db, collection)
	defer ms.Close()
	return mc.Remove(query)
}

func Run(cfg CallerCfg) (c *Client, err error) {
	var (
		m        *mgo.Session
		dialInfo *mgo.DialInfo
	)
	dialInfo = &mgo.DialInfo{
		Addrs:    []string{cfg.URL},
		Source:   cfg.Source,
		Username: cfg.User,
		Password: cfg.Password,
	}
	m, err = mgo.DialWithInfo(dialInfo)
	if err != nil {
		log.Fatalln("create session error ", err)
	}
	mgo.SetLogger(&cLogger{})
	mgo.SetDebug(cfg.Debug)
	// Optional. Switch the session to a monotonic behavior.
	m.SetMode(mgo.Monotonic, true)
	return &Client{cfg, m}, err
}

```

## 使用
```go
/**
* @Author: myxy99 <myxy99@foxmail.com>
* @Date: 2020/7/30 14:34
 */
package mongoDb

import (
	"github.com/globalsign/mgo/bson"
	"testing"
)

type Movie struct {
	Id   bson.ObjectId `bson:"_id" json:"id"`
	Name string        `bson:"name" json:"name"`
}

const (
	db         = "Movies"
	collection = "MovieModel"
)

func TestClient_Insert(t *testing.T) {
	var (
		cfg CallerCfg
		c   *Client
		err error
	)
	cfg = CallerCfg{
		Debug:    true,
		URL:      "127.0.0.1:27017",
		Source:   "admin",
		User:     "",
		Password: "",
	}
	c, err = Run(cfg)
	if err != nil {
		panic(err.Error())
	}
	err = c.Insert(db, collection, Movie{
		Id:   bson.NewObjectId(),
		Name: "test123",
	})
}

```