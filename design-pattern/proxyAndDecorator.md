---
title: 代理模式
comments: true
---

![](https://s2.loli.net/2022/06/13/mHxk1b3sVtUZJdW.png)

Decorator Vs Proxy

我感觉在很大程度上，这两种模式是十分接近且代码实现都差不多的，所以我好奇的是怎么区分以及它们各自的应用场景。（这篇整理是我学习这两种模式后估计留了几天时间写的，现在回顾确实区分不出来了。）

<!--more-->

## 代理模式

`《大话设计模式》` 中有提到的一个例子比较有意思，有个小伙想追班花，但是自己羞于表达，送礼物的时候都是找好兄弟帮忙，时间长了，好兄弟和班花产生了感情，自己白白给别人做了嫁衣。这种通过第三方的方式去（掩盖了实体存在的事实）做事，称为代理模式。下面的是我根据他的定义写了一个 demo：

```go
package main

import "fmt"

type gives interface {
	GiveFlower()
	GiveCandy()
	GiveWater()
}

type Z struct {
}

func (z Z) GiveFlower() {
	fmt.Println("GiveFlower")
}
func (z Z) GiveCandy() {
	fmt.Println("GiveCandy")
}
func (z Z) GiveWater() {
	fmt.Println("GiveWater")
}

type proxy struct {
	gives gives
}

func (p proxy) GiveFlower() {
	p.gives.GiveFlower()
}

func (p proxy) GiveCandy() {
	p.gives.GiveCandy()
}

func (p proxy) GiveWater() {
	p.gives.GiveWater()
}

func NewProxy(real gives) *proxy {
	p := new(proxy)
	p.gives = real
	return p
}

func main() {
	proxy := NewProxy(Z{})
	proxy.GiveFlower()
	proxy.GiveCandy()
	proxy.GiveWater()
}
```



然而在其他地方了解到的代理模式，除了实现相同接口外，还附加了一些额外功能，就比如下面这个例子（来自 Go By Example）：

```go
// server.go 共同接口

package main

type server interface {
    handleRequest(string, string) (int, string)
}
```



```go
// nginx.go 主体

package main

type nginx struct {
    application       *application
    maxAllowedRequest int
    rateLimiter       map[string]int
}

func newNginxServer() *nginx {
    return &nginx{
        application:       &application{},
        maxAllowedRequest: 2,
        rateLimiter:       make(map[string]int),
    }
}

func (n *nginx) handleRequest(url, method string) (int, string) {
    allowed := n.checkRateLimiting(url)
    if !allowed {
        return 403, "Not Allowed"
    }
    return n.application.handleRequest(url, method)
}

func (n *nginx) checkRateLimiting(url string) bool {
    if n.rateLimiter[url] == 0 {
        n.rateLimiter[url] = 1
    }
    if n.rateLimiter[url] > n.maxAllowedRequest {
        return false
    }
    n.rateLimiter[url] = n.rateLimiter[url] + 1
    return true
}
```



```go
// application.go 真实主体

package main

type application struct {
}

func (a *application) handleRequest(url, method string) (int, string) {
    if url == "/app/status" && method == "GET" {
        return 200, "Ok"
    }

    if url == "/create/user" && method == "POST" {
        return 201, "User Created"
    }
    return 404, "Not Ok"
}
```



```go
// main.go 客户端代码

package main

import "fmt"

func main() {

    nginxServer := newNginxServer()
    appStatusURL := "/app/status"
    createuserURL := "/create/user"

    httpCode, body := nginxServer.handleRequest(appStatusURL, "GET")
    fmt.Printf("\nUrl: %s\nHttpCode: %d\nBody: %s\n", appStatusURL, httpCode, body)

    httpCode, body = nginxServer.handleRequest(appStatusURL, "GET")
    fmt.Printf("\nUrl: %s\nHttpCode: %d\nBody: %s\n", appStatusURL, httpCode, body)

    httpCode, body = nginxServer.handleRequest(appStatusURL, "GET")
    fmt.Printf("\nUrl: %s\nHttpCode: %d\nBody: %s\n", appStatusURL, httpCode, body)

    httpCode, body = nginxServer.handleRequest(createuserURL, "POST")
    fmt.Printf("\nUrl: %s\nHttpCode: %d\nBody: %s\n", appStatusURL, httpCode, body)

    httpCode, body = nginxServer.handleRequest(createuserURL, "GET")
    fmt.Printf("\nUrl: %s\nHttpCode: %d\nBody: %s\n", appStatusURL, httpCode, body)
}
```



可以看到 `nginxServer` 除了调用原本的 `handleRequest` 还额外调用了代理本身的一个方法，所以我的疑问也出在这里，装饰器模式可以添加额外的方法，代理模式也添加了额外的方法，都是在不修改原有类的基础上进行修改，那他们具体的区别是？



> 代理模式，注重对对象某一功能的流程把控和辅助。它可以**控制对象做某些事**，重心是为了借用对象的功能完成某一流程，而非对象功能如何。
>
> 装饰模式，**注重对对象功能的扩展**，它不关心外界如何调用，只注重对对象功能的加强，装饰后还是对象本身。



有了这个定义，其实就清晰很多了。



## 动态代理

