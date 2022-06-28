---
title: 参数是怎么传给 goroutine 的
comments: true
---

*go version: 1.16*

![](https://s2.loli.net/2022/06/27/j7xI9vwEdRF8eTB.png)

文章内容接之前的 `variable shadowing`  做了一些延伸，在批量创建 `goroutine` 时，避免不了参数传递，通常的做法如下：

```go
for i := 0; i < 10; i++ {
    go func (i int) {
        println(i)
    }(i)
}
// wait all g done
```

<!--more-->

其实也可以通过 `variable shadowing` 来解决，这两种方法达到的效果是一样的，如下：

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



栈是由*高*地址向*低*地址增长的，有时候不好确定这个增长方向，因为 x86 代码和 go 的汇编存在差异，区分不好应该是从左往右看，还是从右往左，既然这该死的脑子记不住，就找到一个窍门，每次先找 sub 或者 add 指令，通过这种方式来区分从哪边看起。



还有一点需要我们注意的就是，如何理解 `go func(){}()` 的这个函数地址与参数位置的关系，是谁把参数放在了与这个函数位置挨着的地方？为什么要挨着放在别的地方行不行？常规情况下，函数的参数传递形式如下图所示：

![](https://s2.loli.net/2022/06/27/QgtHDkSq75E9sGJ.png)

这里涉及到调用规约的内容，可以参考曹大的文章（文末引路），主要讲的就是函数间参数传递的方式以及返回值放在哪里等。



查看创建 goroutine 的源码，注释中有关于调用规约的描述：

```go
// The stack layout of this call is unusual: it assumes that the
// arguments to pass to fn are on the stack sequentially immediately
// after &fn.
```

首先提到的就是这里的的栈传参形式与平常是不一样的，这里的 `fn` 指的就是 go 关键字后面的 func，`&fn` 的意思就是 fn 函数的地址（一开始以为在执行这个取址操作后..）。



在 `fn` 函数地址后面，就是传递给它的参数。有意思的地方来了，**goroutine 的参数是借用 newproc 传递给了 fn 函数，但是在 newproc 函数签名中只有两个参数**，如下：

```go
func newproc(siz int32, fn *funcval) {
    // ommit
}
```

![](https://s2.loli.net/2022/06/27/VNzjKJUTlZWnf13.png)



这里通过汇编代码验证下，我们增加传递给 goroutine 参数，加到 3 个：

```assembly
# 第一个参数 siz 代表所有参数占用的字节数
mov dword ptr [rsp], 0x18
# 把 fn 的地址 load 出来
lea rbx, ptr [rip+0x2b572]
# 第二个参数 fn
mov qword ptr [rsp+0x8], rbx
# fn 的第一个参数
mov qword ptr [rsp+0x10], rdx
# fn 的第二个参数
mov qword ptr [rsp+0x18], rax
# fn 的第三个参数
mov qword ptr [rsp+0x20], rcx
# 调用 newproc
call $runtime.newproc
```

> 虽然计算机的世界没有 magic，但是感觉是真 tmd 神奇。



最后在创建新 goroutine 的时候，把参数拷贝到当前 goroutine 的栈地址空间。分析的过程没什么难度，死记硬背并不是什么秘诀，掌握寻找答案的方法更重要。



最后，关于调用规约的相关内容，指路[ [曹大博客](https://www.xargin.com/go1-17-new-calling-convention/) ]。