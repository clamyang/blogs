---
title: 并发控制-互斥（自旋锁、互斥锁）
comment: true
---

锁.

<!--more-->

如何定义一把锁？

- basic - 能否达到互斥
- fairness - 能否保证公平，极端情况下，是否会出现 thread 获取不到锁的情况
- performance - 性能，使用锁带来的负载。
  - 没有竞争时
  - 单核多线程竞争
  - 多核多线程竞争

## 原子指令 Bus Lock

正如我们最熟悉的**累加**运算，很多像我一样的初级程序员都会在编写并发程序的时候注意不到这个问题，运算过程中涉及到 load/store 指令，但是并不是原子操作。

所以在没有并发控制的情况下，很容易就会导致 data-race 的问题。我们可以通过算法，比如之前文章提到的 `Peterson` 来实现并发控制，但是我们可以图省事，把这项工作交给硬件工程师去做呀？

我们向硬件工程师提需求，让底层提供一条可以在多线程场景下使用的指令而且还得是并发安全的



有了硬件提供给我们一条原子的 读+写 （load+store） 指令，那我们就放心的去并发访问了，这里就是 exchange （x86 叫这个，别的又叫 Test_And_Set） 指令。

> 请注意！不要误以为是 CAS



只要执行 exchange 每次都会进行**值的交换**，就看交换得到的值是什么！



如何使用 exchange 实现互斥？

`lock xchg` 这个 lock 就会将 bus 锁住（或许这就是为啥它叫 Bus Lock），所以这时候就不是共享的内存了，是**持有这个 bus lock** 线程专有的内存。

```c
// global param
int table = YES;

void lock() {
retry:
  // use NOPE exchange table(YES)
  int got = xchg(&table, NOPE);
  // if got == NOPE mean other thread have exchanged; so should wait
  if (got == NOPE)
    goto retry;
  assert(got == YES);
}

// unlock: use YES exchange NOPE
void unlock() {
  xchg(&table, YES)
}
```

实现互斥的协议（即上述代码的描述）：

- 想上厕所的同学 (一条 xchg 指令)
  - 天黑请闭眼（锁总线）
  - 看一眼桌子上有什么 (🔑 或 🔞)
  - 把 🔞 放到桌上 (覆盖之前有的任何东西)
  - 天亮请睁眼；看到 🔑 才可以进厕所哦
- 出厕所的同学
  - 把 🔑 放到桌上



> 原子指令的模型
>
> - 保证之前的 store 都写入内存（在执行原子指令时，保证所有的 store 都执行完毕）
> - 保证 load/store 不与原子指令乱序



**这里真的是解答了一个困惑我很长时间而且没有得到解决的疑问！**

原子操作并不是无锁的！



### // TODO Lock 指令的现代实现



## RISC-V 中原子操作的实现



## 自旋锁 (spin lock)

[wiki](https://en.wikipedia.org/wiki/Spinlock#:~:text=In%20software%20engineering%2C%20a%20spinlock,a%20kind%20of%20busy%20waiting.) 定义：a **spinlock** is a [lock](https://en.wikipedia.org/wiki/Lock_(computer_science)) that causes a [thread](https://en.wikipedia.org/wiki/Thread_(computer_science)) trying to acquire it to simply wait in a loop ("spin") while repeatedly checking whether the lock is available. 



> spin 自旋，在这里就是循环等待锁被释放；GMP 模型中有一个 spining m 用来寻找可执行的 g 同样也是自旋。



从上述的代码也不难看出，就是通过不断的循环，load 锁的状态，直到锁被上一个持有者释放。

自旋锁存在的问题：

- 空转
- 获得自旋锁的线程可能被切换

自旋锁使用场景：

操作系统内核的并发数据结构

- 临界区几乎不拥堵，很快就能执行完
- 持有自旋锁时禁止执行流切换



## 互斥锁



