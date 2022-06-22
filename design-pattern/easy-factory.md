---
title: 简单工厂模式
comments: true
---

自己照着书上 JAVA 的代码翻译成了 Go 的代码，但是感觉很生硬，就比如，在 Go 中我们获取到了一个对应类型的 interface，这种情况下想要设置对应结构体的字段很麻烦，必须要实现 JAVA 中，那种 Get Set 的东西，如果说不想用这种方式，那么还需要对 interface 进行转换，并不是那么优雅。



工厂模式解决了什么问题：**创建对象的问题**。



<!--more-->

## 计算器的例子

### JAVA 代码如下：

这里展示的并不完整，可以通过 UML 图来看下类与类之间的关系。

![](https://s2.loli.net/2022/06/07/MbtzFSdN4V561Ar.png)

UML 图如下：

![](https://s2.loli.net/2022/06/07/AJlb27BFPM6hyea.png)

### Go 代码如下：

```go
package main

type Operation interface {
    GetResult()
    SetNum
}

type SetNum interface {
    SetNumber1(int)
    SetNumber2(int)
}

// 加法
type SADD struct {
    Number1 int
    Number2 int
}

func (s *SADD) GetResult() {
    println(s.Number1 + s.Number2)
}

func (s *SADD) SetNumber1(val int) {
    s.Number1 = val
}

func (s *SADD) SetNumber2(val int) {
    s.Number2 = val
}

// 减法
type SSUB struct {
    Number1 int
    Number2 int
}

func (s *SSUB) GetResult() {
    println(s.Number1 - s.Number2)
}

func (s *SSUB) SetNumber1(val int) {
    s.Number1 = val
}

func (s *SSUB) SetNumber2(val int) {
    s.Number2 = val
}

// 工厂函数
func OperationFactory(oper string) Operation {
    switch oper {
        case "+":
        return &SADD{}
        case "-":
        return &SSUB{}
    }
    return nil
}

func main() {
    obj := OperationFactory("+")
    obj.SetNumber1(1)
    obj.SetNumber2(1)

    obj.GetResult()
    // Output: 2
}
```



综上所述，我们通过传入不同的运算符，然后返回不同的子类，调用每个子类相应的方法。



## 商场打折例子

比如我们要实现一个结算功能，然后根据商场不同的优惠活动，最终计算出一个总价出来，下图为小菜同学写的初版代码（其实人人都是小菜，但是没有人承认自己是小菜）。

![](https://s2.loli.net/2022/06/07/NenaDz4uSV2xt6E.png)

那么这段代码存在什么问题呢？

1. 重复代码过多，`Convert.ToDouble` 调用多次，而且计算打折活动时的算法都是一样的，只是输入的折数不同。
2. 代码扩展性不好，如果说这时候要增加额外的优惠活动，比如满 300-50，需要改动很多。



用工厂模式重写，通过计算器的例子其实不难想出，我们仍需要一个父类，每个优惠活动都是其子类。

子类的抽象：

- 并不需要为每种活动添加一个类，比如打八折、打九折这种可以抽象成一类
- 满 300 减 50 这种可以抽象成一类

**面向对象的编程，并不是类越多越好，类的划分是为了封装，但分类的基础是抽象，具有相同属性和功能的对象的抽象集合才是类**



重写后的代码结构如下图所示：

![](https://s2.loli.net/2022/06/07/cIM8w3vrBTVGgJU.png)





依次分为了，正常结算、优惠结算、满减结算；然后通过工程类，根据不同的需要创建特定的子类。



至此，一切看起来都很好，但是如果说商家想患者方法忽悠消费者，他又搞了一个打 5 折的活动，这时候难道我们要把响应的代码加上然后重新打包上线嘛？



很显然不能这样做，由此引出**策略模式**。