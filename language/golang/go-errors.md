---
title: go error handling
comments: true
---

![](https://s2.loli.net/2022/06/08/yfjlCg7NJxwhqbu.png)

这篇文章是我学习董哥发布的 [错误处理](https://mytechshares.com/2021/11/22/go-error-best-practice) 文章的总结，好记性不如烂笔头，避坑啦！

<!--more-->



通常情况下打印的错误信息是不包含堆栈的。

```go
package main

import (
	"fmt"
	"io/ioutil"
)

func main() {
    // %v 与 %s 相同
	fmt.Printf("%v\n", readFile())
}

func readFile() error {
	_, err := ioutil.ReadFile("")
	return err
}

// Output: open : The system cannot find the file specified.
```



但是在很多情况下没有堆栈信息是很难定位问题的，除非是 panic 了，但是业务逻辑的代码不能直接 panic 可能会直接导致服务崩溃。



## 官方方法 %w

还是上面那个例子：

```go
func readFile() error {
	_, err := ioutil.ReadFile("")
	// 这里只是简单了 wrap 错误信息，并不会增加调用栈的相关内容
    return fmt.Errorf("%w\n", err)
}

// Output: open : The system cannot find the file specified.
```



## github.com/pkg/errors

```
%s    print the error. If the error has a Cause it will be
      printed recursively.
%v    see %s
%+v   extended format. Each Frame of the error's StackTrace will
      be printed in detail.
```



### errors.Wrap

```go
package main

import (
	"fmt"
	"github.com/pkg/errors"
	"io/ioutil"
)

func main() {
	fmt.Printf("%+v\n", readFile())
}

func readFile() error {
	_, err := ioutil.ReadFile("")
    return errors.Wrap(err, "something failed")
}
/*
open : The system cannot find the file specified.
something failed
main.readFile
	D:/program/code/newProject/errors-dev/main.go:15
main.main
	D:/program/code/newProject/errors-dev/main.go:10
runtime.main
	D:/program/language/go/go1.16.5/src/runtime/proc.go:225
runtime.goexit
	D:/program/language/go/go1.16.5/src/runtime/asm_amd64.s:1371
*/
```

使用 errors.Wrap 可以告诉我们堆栈信息，以及报错内容，还有一点需要注意的是：



> 在调用 errors.Wrap() 时，**最好判断一下 err 是不是 nil**，之前在群里时候就有一个哥们遇到了这个问题，类型断言失败了，err 是 nil，然后进行了 wrap 操作，wrap 中判断 err 是 nil 会直接返回 nil，导致后续代码执行的时候拿到了一个 nil pointer。下图是他当时的场景。

![断言失败，错误为 nil，return nil, nil](https://s2.loli.net/2022/06/08/VBrMAkzKmLN6b5H.png)



董哥在文章中有提到这一点，自己现在还没遇到，既然有人已经在上边踩过坑了，需要引起注意！



还有一点需要注意的就是，**不要过多使用 errors.Wrap** 这样会输出重复的堆栈信息，可以试试下面这段代码：

```go
func readFile() error {
	_, err := ioutil.ReadFile("")
    // 会输出两次堆栈信息
    return errors.Wrap(errors.Wrap(err, "something failed"), "failed!")
}
```



## error 级联问题

这个是没接触过的问题，但是或多或少听说过 interface 赋值的问题（抓个时间把 interface 好好看看），在 Go 中，error 其实就是一个 interface。

```go
// The error built-in interface type is the conventional interface for
// representing an error condition, with the nil value representing no error.
type error interface {
	Error() string
}

/*
  这时我们创建一个结构体，实现这个 interface
*/

type myError struct {
    Info string
}

func (my *myError) Error() string {
    return my.Info
}
```

完整代码如下：

```go
package main

import "fmt"

type myError struct {
    string
}

func (i *myError) Error() string {
    return i.string
}

func Call1() error {
    return nil
}

func Call2() *myError {
    return nil
}

func main() {
    // 在这个问题中我们需要知道的一点就是
    // err 是一个 interface
    err := Call1()
    if err != nil {
        fmt.Printf("call1 is not nil: %v\n", err)
    }

    err = Call2()
    fmt.Println(err==nil)  // false
    fmt.Println(err)	   // nil
    // 上边这两个输出结果真的很矛盾。。
    if err != nil {
        fmt.Printf("call2 err is not nil: %v\n", err)
    }
}
```



答案就是：**[An interface value is equal to `nil` only if both its value and dynamic type are `nil`.](https://yourbasic.org/golang/gotcha-why-nil-error-not-equal-nil/) **接口类型的值只有在 type 和 value 都是空的时候才是 nil，这里我们调用 Call2() 函数后，nil 有了具体的类型，这才导致了后续判断的时候出现奇怪的现象。



不错，又是收获满满的一天，2022-06-08 22:42
