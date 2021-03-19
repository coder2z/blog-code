+++
tags = ["golang", "算法"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang实现雪花算法"
date = "2020-06-16T13:09:56+08:00"
+++


# 雪花算法

## 雪花算法

雪花算法的原始版本是scala版，用于生成分布式ID（纯数字，时间顺序）,订单编号等
    自增ID：对于数据敏感场景不宜使用，且不适合于分布式场景。 GUID：采用无意义字符串，数据量增大时造成访问过慢，且不宜排序。

## 算法描述
* 最高位是符号位，始终为0，不可用。
* 41位的时间序列，精确到毫秒级，41位的长度可以使用69年。时间位还有一个很重要的作用是可以根据时间进行排序。
* 10位的机器标识，10位的长度最多支持部署1024个节点。
* 12位的计数序列号，序列号即一系列的自增id，可以支持同一节点同一毫秒生成多个ID序号，12位的计数序列号支持每个节点每毫秒产生4096个ID序号。


## Golang 实现
```golang
package main

import (
    "errors"
    "fmt"
    "sync"
    "time"
)

const (
    workerBits  uint8 = 10
    numberBits  uint8 = 12
    workerMax   int64 = -1 ^ (-1 << workerBits)
    numberMax   int64 = -1 ^ (-1 << numberBits)
    timeShift   uint8 = workerBits + numberBits
    workerShift uint8 = numberBits
    startTime   int64 = 1525705533000 // 如果在程序跑了一段时间修改了epoch这个值 可能会导致生成相同的ID
)

type Worker struct {
    mu        sync.Mutex
    timestamp int64
    workerId  int64
    number    int64
}

func NewWorker(workerId int64) (*Worker, error) {
    if workerId < 0 || workerId > workerMax {
        return nil, errors.New("Worker ID excess of quantity")
    }
    // 生成一个新节点
    return &Worker{
        timestamp: 0,
        workerId:  workerId,
        number:    0,
    }, nil
}

func (w *Worker) GetId() int64 {
    w.mu.Lock()
    defer w.mu.Unlock()
    now := time.Now().UnixNano() / 1e6
    if w.timestamp == now {
        w.number++
        if w.number > numberMax {
            for now <= w.timestamp {
                now = time.Now().UnixNano() / 1e6
            }
        }
    } else {
        w.number = 0
        w.timestamp = now
    }
    ID := int64((now-startTime)<<timeShift | (w.workerId << workerShift) | (w.number))
    return ID
}
func main() {
    // 生成节点实例
    node, err := NewWorker(1)
    if err != nil {
        panic(err)
    }
    for {
        fmt.Println(node.GetId())
    }
}

```

参考连接：[https://www.cnblogs.com/blogbobo/p/13169714.html](https://www.cnblogs.com/blogbobo/p/13169714.html)
    
