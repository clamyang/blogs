---
title: 策略模式
comments: true
---



由工厂模式引出策略模式，需要思考的是作者是怎么一步步引导我们的，以及工厂模式到底解决了什么问题？

<!--more-->

## 策略模式

二话不说先来看下策略模式的定义是什么？

策略模式（Strategy）：它定义了算法家族，分别封装起来，让它们之间可以互相替换，此模式让算法的变化，不会影响到使用算法的客户。[DP]



由此可以联想到购物 App，618、双十一活动时以及一些法定节假日，这些都是促销活动，每次活动都有不同的优惠力度。商家在改变优惠活动的时候并不影响我们，给我们呈现的就是在那些节日我们拼单、凑满减。可能有时候就想花那 50 块钱，为了凑满减硬生生多花了几百。



言归正传，我们看下策略模式的结构图：

![](https://s2.loli.net/2022/06/07/QPrZjJdivU4EMsl.png)

看到这张图第一眼其实跟工厂模式感觉不到什么差异，只不过是 Context 跟 Strategy 的关系变了。



更具体的区别在于调用时一个传递的是标识（用字符串来区分是哪计算类型），另一个是传递具体的对象，然后调用特定的方法。



![策略模式下的客户端实现](https://s2.loli.net/2022/06/08/IwgpQhSyAPVGdXu.png)



需要初始化哪个类依赖于客户端的选择，那么可以通过哪种方式将算法的选择转移到后端的具体实现中呢？



## 策略模式与工厂模式的结合

![](https://s2.loli.net/2022/06/08/jMgGqYbRroJd2ED.png)

不难发现，这里构造函数接收一个字符串，也就是客户端传过来的值，然后我们根据这个值找到具体的分支（具体的子类），然后将这个子类赋给 Context 类的成员变量，后续客户端直接调用 Context 类的方法即可。



做到了，将类的初始化和客户端解耦。



## 策略模式解析

策略模式是一种定义一系列算法的方法，从概念上来看，所有这些算法完成的都是相同的工作，只是实现不同，它可以以相同的方式调用所有的算法，减少了各种算法类与使用算法类之间的耦合[DPE]。



**优点**：

- 策略模式的Strategy类层次为Context定义了一系列的可供重用的算法或行为。继承有助于析取出这些算法中的公共功能[DP]
- 策略模式的优点是简化了单元测试，因为每个算法都有自己的类，可以通过自己的接口单独测试[DPE]
  - 这个其实很好理解，大家都是独立的互不影响
- 在基本的策略模式中，选择所用具体实现的职责由客户端对象承担，并转给策略模式的Context对象[DPE]
- 策略模式就是用来封装算法的，但在实践中，我们发现可以用它来封装几乎任何类型的规则，只要在分析过程中听到需要在不同时间应用不同的业务规则，就可以考虑使用策略模式处理这种变化的可能性[DPE]



## Golang 策略模式实现

[策略模式代码](https://golangbyexample.com/strategy-design-pattern-golang/) 

```go
// 三种策略
type lru struct {}
func (l *lru) evict() {}

type lfu struct {}
func (l *lfu) evict() {}

type fifo struct {}
func (l *fifo) evict() {}

// 公共接口
type algo interface {
    evict()
}

type cache struct {
    // 具体使用哪个算法
    algo         algo
    // 存储的键值对
    storage      map[string]string
    // 当前容量
    capacity     int
    // 最大容量
    maxCapacity  int
}

func initCache(e evictionAlgo) *cache {
    storage := make(map[string]string)
    return &cache{
        storage:      storage,
        algo:         e,
        capacity:     0,
        maxCapacity:  2,
    }
}

// 设置当前的缓存算法，可以动态修改
func (c *cache) setAlgo(e algo) {
    c.algo = e
}

func (c *cache) add(key, value string) {
    if c.capacity == c.maxCapacity {
        c.evict()
    }
    c.capacity++
    c.storage[key] = value
}

func (c *cache) get(key string) {
    delete(c.storage, key)
}

// 调用对应算法的删除操作
func (c *cache) evict() {
    c.algo.evict(c)
    c.capacity--
}

// 客户端代码如下：
package main

func main() {
    // 首先使用 lfu 进行初始化操作
    lfu := &lfu{}
    cache := initCache(lfu)
    cache.add("a", "1")
    cache.add("b", "2")
    cache.add("c", "3")
    // 这时修改为 lru 的模式，如果要删除的话
    // 使用的就是 lru 算法。
    lru := &lru{}
    cache.setEvictionAlgo(lru)
    cache.add("d", "4")
    fifo := &fifo{}
    // 同上
    cache.setEvictionAlgo(fifo)
    cache.add("e", "5")
}
```



## 什么时候使用策略模式

- 当一个对象需要支持不同的行为并且你想在运行时改变；
- 当你想避免在运行时使用大量条件判断选择算法时；
- 当你有相似的不同算法并且它们仅在执行某些行为的方式上有所不同时。
