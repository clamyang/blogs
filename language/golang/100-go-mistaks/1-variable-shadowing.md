---
title: Variable shadowing
comments: true

---

本文中的内容来自《100 Go Mistakes and How to Avoid Them》，作者总结了常见、易犯的错误，比较适合刚学习 Go 的同学。

<!--more-->

## Variable shadowing

这个问题我最开始实习的时候还真遇到过，不过经历一次之后就记住了。主要问题在于对变量作用域以及 `:=` 的理解：

```go
var client *http.Client
if tracing {
    // 这个 client 是一个新的变量
    client, err := createClientWithTracing()
    if err != nil {
        return err
    }
    // 如果这里不打印的话，会报 declared but not use 的错误
    log.Println(client)
} else {
    client, err := createDefaultClient()
    if err != nil {
        return err
    }
    log.Println(client)
}
// Use client
```

解决方案这里一共有两种：

- 在 if 语句代码块外部声明好（err 也要声明）使用 `=` 号赋值；

  ```go
  var client *http.Client
  var err error
  
  if xx {
      client, err = xx()
  } else {
      client, err = xx1()   
  }
  
  // 代码更加简洁
  if err != nil {
      return err
  }
  ```

- 在 if 语句代码块中使用额外的变量接收返回值，然后对 client 进行赋值

  ```go
  // ...
  c, err := createClientWithTracing()
  if err != nil {
      return err
  }
  client = c
  // ...
  ```



这两种方法都是对的，拿第一种来说，我们减少了额外的变量声明，只执行了一次变量分配，而且在进行错误处理时进行统一。我以前写的代码是第一种，不过针对错误处理，我是在每个代码块中都写了，确实有些啰嗦，这下以后就可以写出更简洁的代码了hh。



## 发生场景

那么我们什么时候会遇到这个问题呢？

**当一个变量的名称，在内部代码块重新声明的时候会发生。**正如我们在上述示例代码中看到的，client 在 if 的代码块又被声明了。



## 如何避免

其实说到这里应该能反应过来，一般这种代码检查工具，go vet 中都应该有，就算没有可能有类似的 linter，通过 `go help vet` 我们就可以看到这样一句话 `For example, the 'shadow' analyzer can be built and run using these commands`

然后我们可以通过 `go install` 命令安装到自己的 GOPATH/bin 路径下，通过 `go vet -vettool=（可执行文件路径）` 就可以检查出来。

```shell
go vet -vettool=C:\Users\bqYang\go\bin\shadow.exe .\main.go

.\main.go:24:3: declaration of "x" shadows declaration at line 22
```

