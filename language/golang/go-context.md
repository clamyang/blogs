---
title: go context
comments: true
---

![](https://s2.loli.net/2022/06/15/4SjfIcJUAtRrigO.png)

这两天在看 Context 的最佳实践，在项目中有用到这个东西，但是又没有实际起作用，只是单纯的作为一个参数传来传去。这篇文章的目的就是学会使用 Context 以及阅读 Context 部分源码实现，还有在使用时需要注意的事项。

<!--more-->

## Context

### Background && TODO

**Background** 一般都是用来当作树的顶点来使用：

```go
func main() {
    // 使用 WithCancel 派生出一个子节点
    // 当前的 Context tree 为：
    // parent --> ctx
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel
    
    // use ctx do something..
}
```

> 需要注意的是，Background 是永远都不会被取消的，这个我们稍后会看到为什么。



**TODO** 一般用在不清楚使用哪个 Context 时：

```go
// 假设我们有个函数需要 context 作为参数
func DoSomething(ctx context.Context, args Args) {}
```

如果说这时候我们不知道要传给这个函数哪个具体的实参，可以使用 `context.TODO` 代替，将来有具体的 Contetxt 可以把这个替换掉。

### 底层实现

在底层这两个其实是同一个东西：

```go
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)
```



既然都说到这了，我们看下源码：

```go
// context 在底层就是一个 interface
type Context interface {
	Deadline() (deadline time.Time, ok bool)
    // 采用了 lazy init 的方式进行初始化
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```

emptyCtx 实现了这个接口，但是与其他 `ctx` 不同的是，emptyCtx 是不会被 cancel 的，也没有任何的 value 和 deadline。



另一个非常有意思的地方，`emptyCtx` 其实是 int 的一个别名，官方注释提到不使用 struct 的原因是**需要不同的地址**，可以动手实验一下：

```go
func printAddr() {    
	type s struct{}
	a := new(s)
	b := new(s)
	fmt.Printf("%p, %p\n", a, b)

    type i int
	i1 := new(i)
	i2 := new(i)
	fmt.Printf("%p, %p\n", i1, i2)
}
/* 
    Output: 
    address of a is 0x8cb1d0, address of b is 0x8cb1d0
    address of i1 is 0xc00000a0a8, address of i2 is 0xc00000a0e0
*/
```

> 为什么不能使用相同的地址？
>
> 个人认为是为了区分 Background 和 TODO，在源码中我们能看到 emptyCtx 有一个方法叫做 String()
>
> ```go
> func (e *emptyCtx) String() string {
>    switch e {
>    case background:
>       return "context.Background"
>    case todo:
>       return "context.TODO"
>    }
>    return "unknown empty Context"
> }
> ```
>
> 如果说我们使用 `type emptyCtx struct{}` 作为 Background 和 TODO 的底层实现，那么在这里打印的时候就会走同一个 case 不能显示正确的输出。
>
> 
>
> 至于其他原因暂时没想到。



同样的 WithCancel / WithValue 等在底层对应的结构体都实现了这个接口。

| funcName                       | structName                                                   |
| ------------------------------ | ------------------------------------------------------------ |
| WithValue()                    | [valueCtx](https://cs.opensource.google/go/go/+/refs/tags/go1.18.3:src/context/context.go;drc=c29be2d41c6c3ed78a76b4d8d8c1c22d7e0ad5b7;l=538) |
| WithDeadline() / WithTimeout() | [timerCtx](https://cs.opensource.google/go/go/+/refs/tags/go1.18.3:src/context/context.go;drc=c29be2d41c6c3ed78a76b4d8d8c1c22d7e0ad5b7;l=465) |
| WithCancel()                   | [cancelCtx](https://cs.opensource.google/go/go/+/refs/tags/go1.18.3:src/context/context.go;drc=c29be2d41c6c3ed78a76b4d8d8c1c22d7e0ad5b7;l=342) |
| Background() / TODO()          | [emptyCtx](https://cs.opensource.google/go/go/+/refs/tags/go1.18.3:src/context/context.go;drc=2580d0e08d5e9f979b943758d3c49877fb2324cb;l=171) |



> 题外话，你看这种方式，像不像之前提到的工厂方法模式，上边表格中的 struct 就是工厂方法子类，如果说我们以后要添加一个新的 ctx 是不影响已存在的内容的，满足了开闭原则。



### WithCancel

```go
// A cancelCtx can be canceled. When canceled, it also cancels any children
// that implement canceler.
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```

**看到这里有一把琐其实就明白为什么 context 可以并发访问**。



cancelCtx 中还有个非常重要的点就是 `cancel` 取消当前的 ctx 以及对应的 child。

```go
// 通过 channel 告诉要 cancel 掉这个 ctx
// cancel 当前节点的子节点
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```

**创建 cancelCtx 过程，源码分析**

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

// use ctx do something
```

```go
// 可以看到，如果 parent 传递为 nil 时，直接panic
// 所以在不知道传什么的时候，也不要传 nil，传 TODO 即可
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := newCancelCtx(parent)
    // 传播这个 ctx，意思为将这个新的 ctx 挂到父节点的下边
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```

```go
// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
    // 父节点的 Done 放在子节点进行初始化操作
    done := parent.Done()
    // 这里就能看到，如果是 emptyCtx， done 直接返回了 nil
    if done == nil {
        return // parent is never canceled
    }


    select {
        // 避免 parent 被取消，额外做一次检查
        case <-done:
        // parent is already canceled
        child.cancel(false, parent.Err())
        return
        default:
    }
	
    // 判断parent的类型，如果是 cancelCtx，走下面这个分支
    // 否则单起一个 goroutine 进行监听
    if p, ok := parentCancelCtx(parent); ok {
        // 拿到父节点的 cancelCtx 后
        // 将子节点给加入进去就行了。
        p.mu.Lock()
        if p.err != nil {
            // parent has already been canceled
            child.cancel(false, p.err)
        } else {
            if p.children == nil {
                p.children = make(map[canceler]struct{})
            }
            p.children[child] = struct{}{}
        }
        p.mu.Unlock()
    } else {
        // 一开始真的想不到什么情况下会走到这个分支
        // 后来想通了，自定义实现的 ctx 就可以走到这里
        // 并且必须实现 Done 方法
        atomic.AddInt32(&goroutines, +1)
        go func() {
            // select 会直接阻塞住，除非下边两个里边的一个“通了”
            select {
                // 如果parent被取消了，取消其子节点
                case <-parent.Done():
                    child.cancel(false, parent.Err())
                // 如果子节点被取消了，不用做额外处理
                case <-child.Done():
            }
        }()
    }
}

// parentCancelCtx 将父节点中的 cancelCtx 提取出啦
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
	done := parent.Done()
    // 验证 parent 是否已经关闭了
	if done == closedchan || done == nil {
		return nil, false
	}
    // 这个 key 就是专门用来判断是否为 cancelCtx 的
    // 一开始还很好奇是在哪里进行存储的这个 key value 的
    // 其实人家根本没存，就是用两个特定的 key 进行比对的
	p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
	if !ok {
		return nil, false
	}
	p.mu.Lock()
	ok = p.done == done
	p.mu.Unlock()
	if !ok {
		return nil, false
	}
	return p, true
}
```



然后另外 `timerCtx` 的实现其实大差不差，增加了时间限制而已。



## 注意事项

使用 Context 需要注意什么呢？

- 不要自己内嵌 `Context`; 取而代之的是显示传递 `Context` 给需要的函数。
- Context 应该作为函数的第一个参数。
- 不要给函数传递 nil 的 `Context`，尽管函数没加限制，使用 `context.TODO` 代替。
- 使用 context Value 附加参数时，只对那些请求范围内的参数使用。



## 最佳实践

参考链接: [Go Context Best Practices](https://www.sobyte.net/post/2022-03/go-ctx-best-practice/) [1]

**使用 context 防止 Goroutine 泄露**，来源于 [[这里](https://pkg.go.dev/context#example-WithCancel)]

```go
package main

import (
	"context"
	"fmt"
)

func main() {
	// gen generates integers in a separate goroutine and
	// sends them to the returned channel.
	// The callers of gen need to cancel the context once
	// they are done consuming generated integers not to leak
	// the internal goroutine started by gen.
	gen := func(ctx context.Context) <-chan int {
		dst := make(chan int)
		n := 1
		go func() {
			for {
				select {
				case <-ctx.Done():
					return // returning not to leak the goroutine
				case dst <- n:
					n++
				}
			}
		}()
		return dst
	}

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel() // cancel when we are finished consuming integers

	for n := range gen(ctx) {
		fmt.Println(n)
		if n == 5 {
			break
		}
	}
}
```

