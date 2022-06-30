---
title: misuse init func
comments: true
---

今天这篇讲的是 `init` 函数使用技巧，平时在人家封装好的代码框架中进行开发，很少独立用到 `init` 函数的地方，其实不小心使用的话坑还是比较多的。

- 一个包中能不能拥有多个 `init` 
- `init` 与全局变量的初始化哪个先执行
- 一个包被导入多次 `init` 是否会执行多次

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

/* 
output:
  var
  init
  main
*/
```



我是倒在了这上边，不过还好，借这个机会来好好了解一下。我们先看下不包含全局变量的情况，在 main 包中引用 redis 包：

![](https://s2.loli.net/2022/06/29/3BlmuwZKvqONH9s.png)



`init ` 函数的执行顺序如上述标号所示，这时候如果一个 package 中有多个 `init` 函数需要执行时，他们的顺序是什么呢？

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



这时候是先执行 `a.go` 中的 `init` 还是 `b.go` 中的呢？ **答案是文件名称排序，谁在前边就先执行谁的 `init`**。

> 所以这里警示我们，务必不能通过文件名称的方式确定 `init` 函数的执行顺序，在不断迭代的过程中文件名称很有可能会被修改。



另一个有意思的地方是，可以在一个文件中写多个 `init` 函数：

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



## 底层实现

```go
// An initTask represents the set of initializations that need to be done for a package.
// Keep in sync with ../../test/initempty.go:initTask
type initTask struct {
	// TODO: pack the first 3 fields more tightly?
	state uintptr // 0 = uninitialized, 1 = in progress, 2 = done
	ndeps uintptr
	nfns  uintptr
	// followed by ndeps instances of an *initTask, one per package depended on
	// followed by nfns pcs, one per init function to run
}

//go:linkname runtime_inittask runtime..inittask
var runtime_inittask initTask

//go:linkname main_inittask main..inittask
var main_inittask initTask	
```

这里我理解 `runtime_inittask` `main_inittask` 一个是 runtime 需要用到的初始化函数，一个是 user program 用到的 init 函数，原因如下：cmd/compile/internal/gc/init.go 文件中有关于 init 的代码（省略了部分代码）：

```go
// fninit makes an initialization record for the package.
func fninit(n []*Node) {
    nf := initOrder(n)

    var deps []*obj.LSym // initTask records for packages the current package depends on
    var fns []*obj.LSym  // functions to call for package initialization

    // Find imported packages with init tasks.
    for _, s := range types.InitSyms {
        deps = append(deps, s.Linksym())
    }

    // Make a function that contains all the initialization statements.
    if len(nf) > 0 {
        ...
    }
    if dummyInitFn.Func.Dcl != nil {
        // We only generate temps using dummyInitFn if there
        // are package-scope initialization statements, so
        // something's weird if we get here.
        Fatalf("dummyInitFn still has declarations")
    }
    dummyInitFn = nil

    // Record user init functions.
    for i := 0; i < renameinitgen; i++ {
        s := lookupN("init.", i)
        fn := asNode(s.Def).Name.Defn
        // Skip init functions with empty bodies.
        if fn.Nbody.Len() == 1 && fn.Nbody.First().Op == OEMPTY {
            continue
        }
        fns = append(fns, s.Linksym())
    }

    // 没有 init 函数需要执行
    if len(deps) == 0 && len(fns) == 0 && localpkg.Name != "main" && localpkg.Name != "runtime" {
        return // nothing to initialize
    }

    // Make an .inittask structure.
    sym := lookup(".inittask")

    ...

    ot := 0
    // 最初状态：未初始化
    ot = duintptr(lsym, ot, 0) // state: not initialized yet
    ot = duintptr(lsym, ot, uint64(len(deps)))
    ot = duintptr(lsym, ot, uint64(len(fns)))
    for _, d := range deps {
        ot = dsymptr(lsym, ot, d, 0)
    }
    
    ...
    // An initTask has pointers, but none into the Go heap.
    // It's not quite read only, the state field must be modifiable.
    ggloblsym(lsym, int32(ot), obj.NOPTR)
}
```



在初始化的过程中是怎么保证一个包不被执行多次的？

inittask 中有一个表示 init 函数状态的变量，0 代表*未执行*， 1 代表*执行中*， 2代表*已完成*。



一个包中有多个 init 函数时，会被加上标号 `pkg.init.0` `pkg.init.1`：

```go
// A function named init is a special case.
// It is called by the initialization before main is run.
// To make it unique within a package and also uncallable,
// the name, normally "pkg.init", is altered to "pkg.init.0".
var renameinitgen int
```