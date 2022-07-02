---
title: 内存相关 API
comments: true
---

这两天在学习内存虚拟化的过程中，发现两个查看内存的两个工具 `free` `pmap` 关于他们的详细描述可以在手册上查阅，这里举两个小例子玩一玩。

<!--more-->

## free

这个比较简单，可以查看当前系统中的内存使用情况：

![](https://s2.loli.net/2022/07/01/uCGFEikarR4WAJQ.png)

## pmap

这个比较有意思，我们可以查看运行中进程的内存布局，`CODE HEAP STACK` ，暂时将布局简单的分成以上三个部分，运行如下代码（采用静待链接的方式编译）：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    // 变换这个数组的大小查看有什么不同
    int *p = (int *)malloc(sizeof(int)*100000);
    printf("%x\n", *p);
    sleep(100);
    return 0;
}
```

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    // 变换这个数组的大小查看有什么不同
    int *p = (int *)malloc(sizeof(int)*100000);
    printf("%x\n", *p);
    sleep(100);
    return 0;
}
```

代码中声明了一个包含**100000**个元素的数组，即使我们现在不知道 `malloc` 会将这个数组分配到哪里也没关系，通过 `pmap` 查看：

```shell
~ # pmap 457
457: ./a.out
0000000000400000       4K r--p  /root/a.out
0000000000401000      12K r-xp  /root/a.out
0000000000404000       4K r--p  /root/a.out
0000000000405000       8K rw-p  /root/a.out
0000000000ab4000     392K rw-p  [heap]
00007fff65583000     132K rw-p  [stack]
00007fff655f3000      16K r--p  [vvar]
00007fff655f7000       4K r-xp  [vdso]
mapped: 572K
```

```shell
~ # pmap 457
457: ./a.out
0000000000400000       4K r--p  /root/a.out
0000000000401000      12K r-xp  /root/a.out
0000000000404000       4K r--p  /root/a.out
0000000000405000       8K rw-p  /root/a.out
0000000000ab4000     392K rw-p  [heap]
00007fff65583000     132K rw-p  [stack]
00007fff655f3000      16K r--p  [vvar]
00007fff655f7000       4K r-xp  [vdso]
mapped: 572K
```

通过观察堆栈的起始地址，可以得出他们的相对位置，看到这个输出就能知道进程内存是如何布局的

- 在最开始的位置就是需要执行的指令，`CODE`，通过前面的 ELF 知识的学习，这里又可以分为 .text .data .bss 段等
- 然后是堆，`HEAP`
- 其次是栈，`STACK`
- `VVAR` `VDSO` 放在下面分开讲

如果说这时候我们还是不清楚数组被分配到了哪里，我们可以尝试减少数组中的元素，然后再查看占用的内存大小。

> 修改后输出的内容都是相同的话，可以尝试禁用优化，在编译的时候加上 `-O0`。

## VDSO

VDSO (Virtual dynamic shared object) ，虚拟动态共享对象，名字就给人一种很难理解的假象。手册中一句话就概述了这个特性是用来做什么的：

```
The "vDSO" (virtual dynamic shared object) is a small shared
library that the kernel automatically maps into the address space
of all user-space applications.
```

**系统调用一定要陷入内核吗？**  VDSO 告诉你不是的，操作系统内核会将某些共享库自动的映射到进程的地址空间。比如在业务逻辑中，很多情况下是需要获取当前时间戳：

![](https://s2.loli.net/2022/07/01/bBUahRS2ZyrwFu3.png)

```shell
1017: /root/a.out
Address           Kbytes     PSS   Dirty    Swap  Mode  Mapping
000055d54f219000       4       4       0       0  r--p  /root/a.out
000055d54f21a000       4       4       4       0  r-xp  /root/a.out
000055d54f21b000       4       0       0       0  r--p  /root/a.out
000055d54f21c000       4       4       4       0  r--p  /root/a.out
000055d54f21d000       4       4       4       0  rw-p  /root/a.out
000055d550af4000       4       0       0       0  ---p  [heap]
000055d550af5000       4       4       4       0  rw-p  [heap]
00007fc662f2e000      84       3       0       0  r--p  /lib/ld-musl-x86_64.so.1
00007fc662f43000     288      18       8       0  r-xp  /lib/ld-musl-x86_64.so.1
00007fc662f8b000     216       3       0       0  r--p  /lib/ld-musl-x86_64.so.1
00007fc662fc1000       4       4       4       0  r--p  /lib/ld-musl-x86_64.so.1
00007fc662fc2000       4       4       4       0  rw-p  /lib/ld-musl-x86_64.so.1
00007fc662fc3000      12       8       8       0  rw-p    [ anon ]
00007ffe9782e000     132      12      12       0  rw-p  [stack]
00007ffe978e5000      16       0       0       0  r--p  [vvar]
00007ffe978e9000       4       0       0       0  r-xp  [vdso]
----------------  ------  ------  ------  ------
total                788      72      52       0
```

`00007ffe978e9733` 正好在 VDSO 的地址范围内，说明这里直接访问了 VDSO 的映射，没有通过系统调用。



## mmap

```c
// 映射，addr 进程中地址，length 区间长度，prot 和 flags 与权限有关，fd 文件描述符
// offset 偏移量；意味着可以把文件的某部分内容映射到进程中。
void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);
int munmap(void *addr, size_t length);

// 修改映射权限
int mprotect(void *addr, size_t length, int prot);
```

文档中有个关于 mmap 的例子还挺不错的 [code example](https://man7.org/linux/man-pages/man2/mmap.2.html)



## malloc &free

`malloc` 用于在堆空间上分配内存的 API，在 C 中堆上的内存分配和释放有开发者来维护。

```c
#include <stdlib.h>

int *x = malloc(sizeof(int));
```



## 常见错误

针对 `malloc` 和 `free` 的常见问题：

**忘记分配内存**

```c
char *src = "hello";
char *dst; // oops! unallocated
strcpy(dst, src); // segfault and die
```

将字符串 `hello` 赋值给变量 dst 时，由于dst指针为指向任何可用的内存，导致失败，正确的做法如下：

```c
char *src = "hello";
char *dst = (char *) malloc(strlen(src)+1); // +1 是给 \0 准备的
strcpy(dst, src);
```



**编译通过并运行不代表是正确的**

虽然我们在控制台看到了正确的输出结果，但也不代表我们的程序是正确的，看下面这个例子：

```c
char *src = "hello";
char *dst = (char *) malloc(strlen(src)); // too small!
strcpy(dst, src); // work properly
```

输出 dst 中的内容时，可以看到 hello 被打印出来了。但是我们分配给变量 dst 的内存大小装不下 src 中的内容。这就会导致，它覆盖了不属于它自己部分。



**忘记初始化刚分配的内存**

C 并不会帮你初始化内存，很有可能你声明了一个整型变量没有进行赋值操作，但当你读他的时候发现是一个神奇怪的数字。



**忘记释放内存**

分配和释放应该成对的出现，这就好像 Go 中打开了一个 response.body 要记得关闭一个道理。如果是一个长时间运行的服务，最终可能把所有的内存耗光，导致 OOM。如果是一个一次性运行的服务，不是放也没什么关系，进程退出了自动会把占有的资源释放了。

> 为什么进程退出了，相关的内存会得到释放？



**还没有使用完内存，就提前释放了**

dangling pointer 问题



**释放已经释放的内存**

double free 的结果是未被定义的，不知道会产生什么结果
