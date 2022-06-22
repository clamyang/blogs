---
title: 真的遇到了变量逃逸
comment: true
---

看看到底是怎么回事吧！

<!--more-->

**我是怎么发现的**

先简单交代一下需要实现的效果，一开始有两个数组，然后通过一个数组中的内容确定是否在另一个数组中存在。一开始我们很容易想到一个时间复杂度为 O（n^2)  算法。

```go
for _, rNode := range remoteNodes {
    for _, lNode := range localNodes {
        if lNode.Name == rNode.Name {
            // do something
        }
    }
}
```



实际工作上代码与这个类似，然后我想这怎么优化一下，降到 O(n) 的时间复杂度，脑子里想到了一种空间换取时间的方式。



额外声明一个 map ，map 我们都熟悉，查询时间复杂度为 O（1），于是就有了如下的代码

```go
nodeMap := make(map[string]NodeStruct, len(remoteNodes))

// 声明好了就开始往里塞数据呗
for _, rNode := range remoteNodes {
    nodeMap[rNode.Name] = &rNode
}
```



当时感觉很牛啊，就开始调试代码，数据也开始往库中插入，最后看一下是否达到了预期。



不看不知道，一看吓一跳，数据都是一样的，也就是说同一条数据插入了 n 次，整的自己有点蒙了。



又开始一行行的进行调试，感觉问题出在 map 上，但是**用 goland ** 调试的时候，看了一下 map 中的内容，是正确的。后边我在 linux 环境下复现的时候，使用 dlv 调试，每次往路边存储一个 key-value 的键值对打印出来的内容都是“错误的”



![image-20220215155911343](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220215155911343.png)

虽然 goland 内置的也是 dlv 但是显示出来的内容确截然不同。



**我是怎么解决的**

至于为什么会发生逃逸，关键操作就是 `nodeMap[rNode.Name] = &rNode` 本来，rNode 是 for 循环内的局部变量，取址操作，让编译器判定这个变量需要存储到堆上。



怎么理解呢，如果说是局部变量，使用完了栈会通过 add 命令加回去，但是，取地址的操作，会使这个变量分配到堆上，后续通过 GC 进行回收。这个可能是最本质的区别。



解决也好办，我们就当它是局部变量就好了，修改代码如下 `nodeMap[rNode.Name] = rNode` 。



再通过 gcflag 看一下，`go run -gcflags="-m" xx.go`

![image-20220215162311639](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220215162311639.png)



直接提示我们 dog 移到了 heap 上..



------

例子代码：

```go
package main

import (
	"bytes"
	"fmt"
	"sync"
)

type dog struct {
	name string
}

func main() {
	dogs := []dog{
		{name: "dog1"},
		{name: "dog2"},
		{name: "dog3"},
		{name: "dog4"},
	}

	dogMap := make(map[string]*dog)

	for _, dog := range dogs {
		dogMap[dog.name] = &dog
	}

	for _, v := range dogMap {
		fmt.Println(v)
	}
}
```

