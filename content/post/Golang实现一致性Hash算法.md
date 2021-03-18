+++
tags = ["golang", "算法", "一致性Hash", "分布式"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang实现一致性Hash算法"
date = "2020-07-09T19:49:48+08:00"
+++


# 什么是一致性Hash算法

一致性Hash算法是使用取模的方法，一致性Hash算法是对2^32取模，什么意思呢？简单来说，一致性Hash算法将整个哈希值空间组织成一个虚拟的圆环，如假设某哈希函数H的值空间为0-2^32-1（即哈希值是一个32位无符号整形），整个哈希环如下：

![](https://gitee.com/coder2m/pic/raw/master/img/20200709195951.png)

圆环的正上方的点代表0，0点右侧的第一个点代表1，以此类推，2、3、4、5、6……直到2^32-1,也就是说0点左侧的第一个点代表2^32-1 

我们把这个由2的32次方个点组成的圆环称为hash环。

假设我们有3台缓存服务器，服务器A、服务器B、服务器C，那么，在生产环境中，这三台服务器肯定有自己的IP地址，我们使用它们各自的IP地址进行哈希计算，使用哈希后的结果对2^32取模，可以使用如下公式示意

> hash(服务器A的IP地址) %  2^32

通过上述公式算出的结果一定是一个0到2^32-1之间的一个整数，我们就用算出的这个整数，代表服务器A，既然这个整数肯定处于0到2^32-1之间，那么，上图中的hash环上必定有一个点与这个整数对应，而我们刚才已经说明，使用这个整数代表服务器A，那么，服务器A就可以映射到这个环上，用下图示意

![](https://gitee.com/coder2m/pic/raw/master/img/20200709201526.png)

同理，服务器B与服务器C也可以通过相同的方法映射到上图中的hash环中。所以现在的hash环：

![](https://gitee.com/coder2m/pic/raw/master/img/20200709201713.png)

这样我们的服务器就映射到了hash环上，现在我们同理也就可以把我们需要访问的对象也放在hash环上。

假设我们现在需要分别在三个服务器上放缓存的文件，我们就使用文件的名字作为计算hash的key：

> hash(文件名) %  2^32

这样也能计算一个值，也就能映射到对应的hash环上。现在我们的hash环就变成了这个样子：

![](https://gitee.com/coder2m/pic/raw/master/img/20200709202336.png)

计算出来了文件的hash值，下一步就是觉得那个服务器存储这个对象了。这里规定是：从计算出来的位置开始向顺时针方向遇到的第一个服务器，进行存储。所以这里文件就会存储在服务器B上。

这就是一致性hash算法。

# 优点

这里我们假设，我们的服务器B突然失效了,我们上面例子中的文件就会存储到服务器c中，这样就算是缓存失效。但是这里服务器失效的是A,就对上面例子中的文件不会有任何的影响。使用一致性哈希算法时，服务器的数量如果发生改变，并不是所有缓存都会失效，而是只有部分缓存会失效，前端的缓存仍然能分担整个系统的压力，而不至于所有压力都在同一时间集中到后端服务器上。这就是一致性哈希算法所体现出的优点。

# hash环的偏斜

理想情况下我们的3个服务器是如上图所示，均匀的分布在hash环上，但是理想往往和现实差距很大：

![](https://gitee.com/coder2m/pic/raw/master/img/20200709203438.png)

实际上映射中，服务器可能会被映射成这样：

![](https://gitee.com/coder2m/pic/raw/master/img/20200709203659.png)

这种情况下缓存的对象很有可能大部分集中缓存在某一台服务器。这就很难受了。这就是hash环的偏斜。那么，我们应该怎样防止hash环的偏斜呢？

# 虚拟节点

虚拟节点就是来解决hash环偏斜的问题的。顾名思义就是在hash环上创建每个服务器的副本，是"实际节点"（实际的物理服务器）在hash环上的复制品,一个实际节点可以对应多个虚拟节点，创建副本后的hash环就变成了这样：

![](https://gitee.com/coder2m/pic/raw/master/img/20200709204300.png)


这样就解决了偏移问题。虚拟节点越多，hash环上的节点就越多，缓存被均匀分布的概率就越大。

# Golang中实现hash环

```golang
package hash

import (
	"errors"
	"hash/crc32"
	"sort"
	"strconv"
	"sync"
)

//声明新切片类型
type units []uint32

//返回切片长度
func (x units) Len() int {
	return len(x)
}

//比对两个数大小
func (x units) Less(i, j int) bool {
	return x[i] < x[j]
}

//切片中两个值的交换
func (x units) Swap(i, j int) {
	x[i], x[j] = x[j], x[i]
}

//创建结构体保存一致性hash信息
type ConsistentHash struct {
	//hash环，key为哈希值，值存放节点的信息
	circle map[uint32]string
	//已经排序的节点hash切片
	sortedHashes units
	//虚拟节点个数，用来增加hash的平衡性
	VirtualNode int
	//map 读写锁
	sync.RWMutex
}

//创建一致性hash算法结构体，设置默认节点数量
func NewConsistent(nodeNum int) *ConsistentHash {
	return &ConsistentHash{
		//初始化变量
		circle: make(map[uint32]string),
		//设置虚拟节点个数
		VirtualNode: nodeNum,
	}
}

//自动生成key值
func (c *ConsistentHash) generateKey(element string, index int) string {
	//副本key生成逻辑
	return element + strconv.Itoa(index)
}

//获取hash位置 计算key 在hash环中对应的位置
func (c *ConsistentHash) hashKey(key string) uint32 {
	//当长度不够填充
	if len(key) < 64 {
		//声明一个数组长度为64
		var tmpList [64]byte
		//拷贝数据到数组中
		copy(tmpList[:], key)
		//使用IEEE 多项式返回数据的CRC-32校验和
		return crc32.ChecksumIEEE(tmpList[:len(key)])
	}
	return crc32.ChecksumIEEE([]byte(key))
}

//更新排序，方便查找 因为后面我们使用的是sort.Search进行查找 sort.Search使用的是二分法进行查找，所以这里需要排序
func (c *ConsistentHash) updateSortedHashes() {
	hashes := c.sortedHashes[:0]
	//判断切片容量，是否过大，如果过大则重置
	if cap(c.sortedHashes)/(c.VirtualNode*4) > len(c.circle) {
		hashes = nil
	}

	//添加hashes
	for k := range c.circle {
		hashes = append(hashes, k)
	}

	//对所有节点hash值进行排序，
	//方便之后进行二分查找
	sort.Sort(hashes)
	//重新赋值
	c.sortedHashes = hashes
}

//向hash环中添加节点
func (c *ConsistentHash) Add(element string) {
	//加锁
	c.Lock()
	//解锁
	defer c.Unlock()
	c.add(element)
}

//添加节点
func (c *ConsistentHash) add(element string) {
	//循环虚拟节点，设置副本
	for i := 0; i < c.VirtualNode; i++ {
		//根据生成的节点添加到hash环中
		c.circle[c.hashKey(c.generateKey(element, i))] = element
	}
	//更新排序
	c.updateSortedHashes()
}

//删除节点
func (c *ConsistentHash) remove(element string) {
	for i := 0; i < c.VirtualNode; i++ {
		delete(c.circle, c.hashKey(c.generateKey(element, i)))
	}
	c.updateSortedHashes()
}

//删除一个节点
func (c *ConsistentHash) Remove(element string) {
	c.Lock()
	defer c.Unlock()
	c.remove(element)
}

//顺时针查找最近的节点
func (c *ConsistentHash) search(key uint32) int {
	//查找算法
	f := func(x int) bool {
		return c.sortedHashes[x] > key
	}
	//使用"二分查找"算法来搜索指定切片满足条件的最小值
	i := sort.Search(len(c.sortedHashes), f)
	//如果超出范围则设置i=0
	if i >= len(c.sortedHashes) {
		i = 0
	}
	return i
}

//根据数据标示获取最近的服务器节点信息
func (c *ConsistentHash) Get(name string) (string, error) {
	//添加锁
	c.RLock()
	//解锁
	defer c.RUnlock()
	//如果为零则返回错误
	if len(c.circle) == 0 {
		return "", errors.New("hash环没有数据")
	}
	//计算hash值
	key := c.hashKey(name)
	i := c.search(key)
	return c.circle[c.sortedHashes[i]], nil
}

```

这样就很简单的实现了golong的一致性hash。

> 这里提一下，为什么是2^32呢?因为这个算法的出现就算为了解决分布式问题，所以在分布式中基本上存储的就算服务器的ip，IPv4的地址是4组8位2进制数组成，所以用2^32可以保证每个IP地址会有唯一的映射。

参考连接：[https://www.cnblogs.com/williamjie/p/9477852.html](https://www.cnblogs.com/williamjie/p/9477852.html)

