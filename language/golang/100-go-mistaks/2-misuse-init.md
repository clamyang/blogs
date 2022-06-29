---
title: misuse init func
comments: true
---

今天这篇讲的是 `init` 函数使用技巧，平时在人家封装好的代码框架中进行开发，很少独立用到 `init` 函数的地方。还有一点就是是否清晰全局变量跟 `init` 函数的执行顺序。感觉这个可以当作一道面试题..

<!--more-->

参考如下代码，你能否说出输出的顺序：

```go
package main
import "fmt"
var a = func() int {
    fmt.Println("var")
    return 0
}()
func init() {
    fmt.Println("init")
}
func main() {
    fmt.Println("main")
}
```



```go
// output:
/* 
 var
 init
 main
*/
```



我是倒在了这上边，不过还好，我们借这个机会来好好了解一下。

不着急，我们先看下不包含全局变量的情况，在 main 包中引用 redis 包：

![image-20220629160509687](C:\Users\ybq28\AppData\Roaming\Typora\typora-user-images\image-20220629160509687.png)

init 函数的执行顺序如上述标号所示，这时候如果一个 package 中有多个 `init` 函数需要执行时，他们的顺序是什么呢？

假设我们有如下的目录结构，每个文件中都有 `init` 函数：

```go
hello
|--a.go
|--b.go
```

```go
// a.go
func init() {
    println("a.go")
}

// b.go
func init() {
    println("b.go")
}

// main.go package main
import _ "hello"

func init() {
    println("main.go")
}

func main() {
    
}
```



这时候是先执行 `a.go` 中的 `init` 还是 `b.go` 中的呢？**答案是文件名称排序，谁在前边就先执行谁的 `init`**。

> 无比不能通过文件名称的方式确定 `init` 函数的执行顺序，在不断迭代的过程中文件名称很有可能会被修改。



另一个有意思的地方是，可以在一个文件中调用多次 `init` 函数。

```go
package main

func init() {
    println("first init")
}

func init() {
    println("second init")
}

func main() {}
```



 `init` 会带来什么样的问题呢？

```go
var db *sql.DB
func init() {
    dataSourceName := os.Getenv("MYSQL_DATA_SOURCE_NAME")
    d, err := sql.Open("mysql", dataSourceName)
    if err != nil {
        log.Panic(err)
    }
    err = d.Ping()
    if err != nil {
        log.Panic(err)
    }
    db = d
}
```

通过 `init` 进行数据库连接的初始化，这里存在至少三个问题：

- 错误处理的局限性，因为 `init` 是没有参数和返回值的。
- 全局变量的可能会在其它的package中被修改。
- 单元测试的局限性，针对这种 `init` 函数，我们想进行单元测试不太可能，在执行用例的时候 `init` 函数已经执行完成了。

**疑问**

假设我们有 `package A`，`package B`，`package main` 每个包都有自己的  `init` 函数，那么他们被导入多次的时候  `init` 会被执行多次吗？

B 中引用了 A，main 中引用了 B，A；我以为 A 的 `init` 会被执行两次，又 tm 被打脸了。 不过比较及时，要不面试的时候就尴尬了。