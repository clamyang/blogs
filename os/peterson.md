---
title: perterson 算法学习
comment: ture
---

并发控制算法之-Peterson

<!--more-->

### Peterson 算法解决了什么问题

- peterson 算法解决的是双线程访问共享变量导致的 data-race 问题。

> 这里确实是双线程，但是该算法在多线程场景下也适用。

### Peterson 算法描述

抽象的代码描述，内容来自 [wiki](https://en.wikipedia.org/wiki/Peterson%27s_algorithm)：

```c
bool flag[2] = {false, false};
int turn;

/*pc 1*/P0:      flag[0] = true;
/*pc 2*/P0_gate: turn = 1;
/*pc 3*/         while (flag[1] == true && turn == 1)
                 {
                     // busy wait
                 }
                 // critical section
                 ...
                 // end of critical section
/*pc 4*/         flag[0] = false;


/* pc 1*/P1:      flag[1] = true;
/* pc 2*/P1_gate: turn = 0;
/* pc 3*/         while (flag[0] == true && turn == 0)
                 {
                     // busy wait
                 }
                 // critical section
                 ...
                 // end of critical section
/* pc 4*/        flag[1] = false;
```

代码解释：

- flag 用于标识哪个线程想要进入临界区
- turn 用于辅助flag表示，也是用来判断是否可以进入临界区



规定如下：

> 如果对方的 flag 为真且 turn 不是自己的名字，需要等待，否则（上述两条件任意一个为假）进入临界区。



这时候我们启动 P1,P2 两个线程，模拟两个线程交替执行，然后将涉及到的状态变化画出来，正如图中所表示的一样。

![peterson](https://s2.loli.net/2022/04/19/l4dy1wmXIgMuUCx.png)

综上所述，Peterson 算法，巧妙地运用了 flag, turn 标识达到了互斥访问的目的。

### 表面上的谦让

这里我们可以看到，如果说 P1 先将 P2 的名字放到了 turn 上，假惺惺的说，你先进吧。但是 P2 执行的时候会覆盖掉 turn，让 P1 进，所以就出现了先修改 turn 变量的线程先进入，后修改的等待。
