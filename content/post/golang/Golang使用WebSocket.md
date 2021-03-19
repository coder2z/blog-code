+++
tags = ["golang", "WebSocket"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang使用WebSocket"
date = "2020-07-17T11:19:32+08:00"
+++



# Golang使用WebSocket

WebSocket是一种通信协议，旨在改善HTTP作为无状态协议通信的效率问题，WebSocket是客户端与服务器之间的全双工连接，客户端和服务器只需要建立一次连接就可以使用该连接进行通信。在我们的项目中，一般客户端是前端页面，使用JavaScript创建WebSocket与后端的WebSocket服务端进行通信。

## 安装依赖

```go get -u github.com/gorilla/websocket```


## 封装方法

```golang
package websocketConn

import (
	"errors"
	"github.com/gorilla/websocket"
	"sync"
)

type Websocket struct {
	conn      *websocket.Conn
	inChan    chan []byte
	ontChan   chan []byte
	closeChan chan byte

	mutex    sync.Mutex
	isClosed bool
}

func InitConnection(wsConn *websocket.Conn) (conn *Websocket, err error) {
	conn = &Websocket{
		conn:      wsConn,
		inChan:    make(chan []byte, 1000), //接受消息管道
		ontChan:   make(chan []byte, 1000), //输出消息管道
		closeChan: make(chan byte, 1),      //关闭通信管道
	}
	// 启动读协程
	go conn.readLoop()
	// 启动写协程
	go conn.writeLoop()
	return
}

func (w *Websocket) ReadMessage() (data []byte, err error) {
	select {
	case data = <-w.inChan:
	case <-w.closeChan:
		err = errors.New("连接已被关闭")
	}
	return
}

func (w *Websocket) WriteMessage(data []byte) (err error) {
	select {
	case w.ontChan <- data:
	case <-w.closeChan:
		err = errors.New("连接已被关闭")
	}
	return
}

func (w *Websocket) Close() {
	w.mutex.Lock()
	defer w.mutex.Unlock()
	if !w.isClosed {
		w.conn.Close()
		close(w.closeChan)
		w.isClosed = true
	}
}

// 获取 发现消息管道中的数据，发送消息
func (w *Websocket) readLoop() {
	var (
		data []byte
		err  error
	)
	for {
		if _, data, err = w.conn.ReadMessage(); err != nil {
			w.Close()
			return
		}
		select {
		case w.inChan <- data:
		case <-w.closeChan:
			w.Close()
			return
		}
	}
}

// 获取 发现消息管道中的数据，发送消息
func (w *Websocket) writeLoop() {
	var (
		data []byte
	)
	for {
		select {
		case data = <-w.ontChan:
		case <-w.closeChan:
			w.Close()
			return
		}
		if err := w.conn.WriteMessage(websocket.TextMessage, data); err != nil {
			w.Close()
			return
		}
	}
}

```


## 使用

```golang
package main

import (
	"github.com/gin-gonic/gin"
	"github.com/gorilla/websocket"
	websocketConn "goStudy/go-webSocket/websocket"
	"net/http"
)

var upGrader = websocket.Upgrader{
	//允许跨域
	CheckOrigin: func(r *http.Request) bool {
		return true
	},
}

func webSocket(c *gin.Context) {
	//升级get请求为webSocket协议
	ws, err := upGrader.Upgrade(c.Writer, c.Request, nil)
	if err != nil {
		return
	}
	conn, err := websocketConn.InitConnection(ws)
	defer func() {
		conn.Close()
	}()
	for {
		data, _ := conn.ReadMessage()
        //接受到的消息处理
		_ = conn.WriteMessage(data)
	}
}

func main() {
	app := gin.New()
	app.GET("/ws", webSocket)
	_ = app.Run(":8888")
}

```

参考连接：[https://github.com/gorilla/websocket](https://github.com/gorilla/websocket)