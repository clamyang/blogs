---
title: 门面模式（外观模式）
comments: true
---

有两周时间没看设计模式内容了，前边学的都忘记差不多了..今天要学的是外观模式。（总感觉将这个英文翻译过来有点词不达意）

<!--more-->

 ## 门面模式

门面模式（Facade），为子系统中的一组接口提供一个一致的界面，此模式定义了一个高层接口，这个接口使得这一子系统更加容易使用。体现了**依赖倒转**与**迪米特法则**，抽象出来一个中间层。

![](https://s2.loli.net/2022/07/05/2WPfhFCj6yUpIHV.png)

最近在学习 K8S 持久化存储的相关内容，这之中也有一个类似的实现。K8S 的 PV 对象对底层实际的存储技术进行封装，再通过 PVC 对 PV 对象进行引用，达到了 K8S 与底层存储技术的解耦。



用一个用户的登录注册的例子可能比较形象，假设现在如左边一样，登录时候调用登录接口，注册时候调用注册接口。现在要增加一个功能，使用手机号的登录时候检查是否存在，存在直接登录，不存在自动注册再登录，例子来源 [[这里](https://github.com/mohuishou/go-design-pattern/tree/master/09_facade)]：

![](https://s2.loli.net/2022/07/05/78QuU9nEI6HPFco.png)



## 代码实现

```go
type user struct {
    username string
}

func (u *user) Login() {
    println("user login")
}

func (u *user) Register() {
    println("user register")
}

type facadeUser struct {
    user
}

func (f *facadeUser) LoginOrRegister() {
    // check user exist
    var exist bool
    if !exist {
        f.user.Register()
    }

    f.user.Login()
}
```

这里实现与上述链接中的些许不同，区别在于**facade 的职责**， **facade 负责的是中转，他的方法中最好不要有具体实现，通过 facade 调用子类的方法或者是将子类的方法再进行封装使其更容易服用。** 通过结构体内嵌可以实现，也可以将user抽象出来一个接口：

```go
type IUser interface {
    Login()
    register
}
```

再将这个接口内嵌到 facade 中：

```go
type facadeUser struct {
    user IUser
}
```



另一个更详细的例子请看这里 [[facade](https://golangbyexample.com/facade-design-pattern-in-golang/)]。