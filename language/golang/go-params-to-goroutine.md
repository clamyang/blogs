---
title: 参数是怎么传给 Goroutine 的
comments: true
---

*go version: 1.16*

文章内容接上文 `variable shadowing` ，做了一点延伸，我们在批量创建 `goroutine` 时，避免不了参数传递。通常的做法如下：

```go
for i := 0; i < 10; i++ {
    go func (i int) {
        println(i)
    }(i)
}
// wait all g done
```

<!--more-->

其实也可以通过 `variable shadowing` 来解决，这两种方法达到的效果是一样的

```go
for i := 0; i < 10; i++ {
    i := i
    go func () {
        println(i)
    }()
}
// wait all g done
```



## 疑问

**由此，我产生了一个疑问，参数是如何传递给 `goroutine` 的**。

## 寻找答案

我们使用上述代码调试，在源码中寻找答案，其实关键的代码就这两行：

```go
func newproc(siz int32, fn *funcval) {
    // fn 地址 再加 8 个字节（跟机器有关，32位4字节），
    // 就是第一个参数的位置，siz 代表字节数，传进来的参数大小
    argp := add(unsafe.Pointer(&fn), sys.PtrSize)
    // ommit
}
```



通过汇编内容，可以发现栈是由*高*地址向*低*地址增长的。（有时候看这个x86代码和go的代码总区分不好应该是从左往右看，还是从右往左，然后每次先找 sub 或者 add 指令区分一下，通过这种方式来区分栈的增长方向）。

![image-20220627160419951](C:\Users\ybq28\AppData\Roaming\Typora\typora-user-images\image-20220627160419951.png)

其实关键在于怎么理解 `go func(){}()` 的这个函数地址与参数位置的关系，是谁把需要的参数放在了与这个函数位置挨着的地方？为什么要挨着放在别的地方行不行？



涉及到调用规约的内容，可以参考大佬们的文章，主要讲的就是函数间参数传递的方式以及返回值放在哪里等。源码中有关于这个约定的描述：

```go
// The stack layout of this call is unusual: it assumes that the
// arguments to pass to fn are on the stack sequentially immediately
// after &fn.
```

这里的 fn 指的就是 go 关键字后面跟的 func，`&fn` 的意思就是 fn 函数的地址（一开始以为在执行这个取址操作后..）。在 fn 函数地址之后，就是传递给它的参数。



最后在创建新 g 的时候，把参数拷贝到当前g的栈地址空间。通篇看下来需要我们理解的就一个点，调用规约，指路[ [曹大博客](https://www.xargin.com/go1-17-new-calling-convention/) ]。