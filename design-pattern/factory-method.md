---
title: 工厂方法模式
comments: true
---

今天学习的另一个创造类的设计模式，工厂方法模式。工厂方法模式是 [简单工厂方法](https://bqyang.top/2022/design-pattern/easy-factory/) 的优化版本，之前我们提到过开闭原则，对扩展开放、对修改封闭。简单工厂方法打破了这个原则，每次增加新的类别时都需要修改工厂类。

<!--more-->

## 工厂方法

正如开头提到的，工厂方法模式比简单工厂模式好的点就在于没有破坏开闭原则。那么它是怎么实现的呢？



简单工厂模式下的 Factory 实现：

```go
// 公共方法抽象出来的接口
type Operation interface {
	GetResult()
	SetNumber1(int)
	SetNumber2(int)
}

// 创建工厂
func OperationFactory(oper string) Operation {
	switch oper {
	case "+":
		return &SADD{}
	case "-":
		return &SSUB{}
	}
	return nil
}
```

这时要添加乘法运算，需要创建一个乘法结构体并实现`Operation`接口，然后需要在创建工厂中，添加相应的分支。为了解决这种紧耦合的方式，工厂方法模式应运而生，工厂方法克服了简单工厂违背开放-封闭原则的缺点，又保持了封装对象创建过程的优点。



> 工厂方法模式（Factory Method），定义一个用于创建对象的接口，让子类决定实例化哪一个类。工厂方法使一个类的实例化延迟到其子类。[DP]

![](https://s2.loli.net/2022/06/14/clzZ1retGRO8IWu.png)

代码实现：

```go
type IOperationFactoryMethod interface {
	CreateOperation() Operation
}

type AddFactory struct {
}

func (addf AddFactory) CreateOperation() Operation {
	return new(SADD)
}

type SubFactory struct {
}

func (subf SubFactory) CreateOperation() Operation {
	return new(SSUB)
}
```



这时候获取实例的方式就改变了，通过具体的工厂子类实现实例的初始化。

```go
// 工厂方法模式
virtualObj := new(AddFactory)
realObj := virtualObj.CreateOperation()
realObj.SetNumber1(200)
realObj.SetNumber2(100)
realObj.GetResult()
```



工厂方法模式实现时，客户端需要决定实例化哪一个工厂来实现运算类（这一点很好理解，在简单工厂中，实例化是通过后端代码逻辑实现的，客户端传给我们对应的符号即可），选择判断的问题还是存在的，也就是说，工厂方法把简单工厂的内部逻辑判断移到了客户端代码来进行。