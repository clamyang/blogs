---
title: free space management
comments: true
---

复习几种空闲内存空间管理的算法，简单总结了 ptmalloc2 tcmalloc 的基本实现。

<!--more-->

## [Best Fit](https://www.geeksforgeeks.org/program-for-best-fit-algorithm-in-memory-management-using-linked-list/?ref=rp)

遍历整个链表，找到一个能满足该内存分配请求的最小内存块。

## [Worst Fit](https://www.geeksforgeeks.org/worst-fit-allocation-in-operating-systems/)

同 Best Fit 类似，遍历整个链表，但是它需要找一个能满足条件中最大的内存块。

## [First Fit](https://www.geeksforgeeks.org/program-first-fit-algorithm-memory-management/?ref=rp)

每次遍历都从链表头部开始，找到一个能满足的就停止遍历。

## [Next Fit](https://www.geeksforgeeks.org/program-for-next-fit-algorithm-in-memory-management/?ref=lbp)

该算法是 First Fit 的优化版本，本次开始遍历的位置是从上一次分配后的位置开始寻找，并不是每次都从头开始。

![](https://s2.loli.net/2022/07/20/cLth1UAKdCnXzwi.png)



## Segregated Lists

这种方式的核心点在于：如果一个应用申请内存的时候总是那几个大小，就维护一个固定大小的链表，专门负责这种大小内存的申请释放，其他大小的内存分配走别的分配器。

问题就是怎么确定一个确定这个 segregated list 中每个内存块是多大的呢？

这里可以参考 Go 中内存内配的实现，把内存分成了 tiny small large；

## Buddy Allocation

![](https://s2.loli.net/2022/07/20/3AHSUYG5gPD1izw.png)

分配和合并的过程都是递归的，free 后会执行相邻内存块的合并，将分配的过程反着执行。



这里有一个 Simulation 文件 [[malloc.py](https://github.com/remzi-arpacidusseau/ostep-homework/tree/master/vm-freespace)]，实现了部分上述算法，可以自己算下 Alloc 后的 freelist 是什么样，检验一下是否真的理解了。



## ptmalloc2 & tcmalloc

ptmalloc2 是 glibc 中默认的分配器，tcmalloc 是 google 推出的内存分配器，go 的内存分配器的同 tcmalloc 类似，但是细节上存在差异。

### ptmalloc2

ptmalloc2 中三种主要的数据结构：

- **malloc_state** 对应 arena header，一个线程的 arena 可以对应多个 heap。每个 heap 有自己的 heap header，但是只有一个 arena header。
- **malloc_chunk** 对应 chunk header，heap 根据我们的 request 被分割成多个 chunk。每个 chunk 都有一个 chunk header ，free 的时候使用的就是这个 header 来计算出要释放的内存大小。
- **heap_info** 对应 heap header，每个 heap 都有自己的 header，*为什么需要多个 heap？* 开始的时候每个线程的 arena 只包含一个 heap，当这个 heap 空间被用尽的时候，就会申请一块新的 heap （与当前的 heap 并不是连续的空间）映射到 arena 中。

用语言描述出来比较抽象，看下面这张图应该就清晰了：

![](https://s2.loli.net/2022/07/22/kife1JDTPlSa9vZ.png)

ptmalloc2 中使用 arena 管理整个 heap，arena 中还包括了 fasetbin,bins,top chunk 等

> main arena 中没有多个 heap 存在也就不存在 heap info，当 heap 中堆内存不够的时候，使用的 brk 系统调用扩展。

**一个线程中有多个 heap 的情况**

![](https://s2.loli.net/2022/07/22/HfcCz96KSabvNJR.png)