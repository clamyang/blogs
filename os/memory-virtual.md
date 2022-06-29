---
title: 内存虚拟化
comments: true
---

这两天在学习内存虚拟化的过程中，发现两个查看内存的两个工具 `free` `pmap` 关于他们的详细描述可以在手册上查阅，这里举两个小例子玩一玩。



## free

这个比较简单，可以查看当前系统中的内存使用情况：

![](C:\Users\ybq28\AppData\Roaming\Typora\typora-user-images\image-20220629143658877.png)



## pmap

这个比较有意思，我们可以查看运行中进程的内存布局，`CODE HEAP STACK` ，我们简单的将布局分成以上三个部分，运行如下代码：

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

我们声明了一个包含**100000**个元素的数组，即使我们现在不知道 `malloc` 会将这个数组分配到哪里也没关系。执行后，我们通过 `pmap` 查看：

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

通过观察堆栈的起始地址，可以得出他们的相对位置，看到这个输出就可以说明进程中的内存是如何布局的，

- 在最开始的位置就是需要执行的指令，`CODE`
- 然后就是堆，`HEAP`
- 其次就是栈，`STACK`
- 其他的可以暂时忽略；

如果说这时候我们还是不清楚数组被分配到了哪里，我们可以尝试减少数组中的元素，然后再查看占用的内存大小。

> 如果说你每次输出的内容都是相同的，可以尝试禁用优化编译的时候加上 `-O0`。