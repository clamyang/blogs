---
title: 宽松内存模型
comment: true
---

---

最近学习到一个关于内存模型的知识点，一开始对内存模型的认识还是以为是描述数据结构实在内存中怎样进行存储的。

<!--more-->

后来发现我这种理解完全被这个名字所误导了，宽松内存模型指的是，在多核处理器的情况下，对访问共享内存行为的描述。



解释下面这个 C 代码：

```c
#include "thread.h"

// 全局变量 x, y
int x = 0, y = 0;

atomic_int flag;
#define FLAG atomic_load(&flag)
#define FLAG_XOR(val) atomic_fetch_xor(&flag, val)
#define WAIT_FOR(cond) while (!(cond)) ;

// 指定编译的时候不要进行内联优化
 __attribute__((noinline))
void write_x_read_y() {
  int y_val;
  // 汇编实现变量 +1 的操作
  asm volatile(
    "movl $1, %0;" // x = 1
    "movl %2, %1;" // y_val = y
    : "=m"(x), "=r"(y_val) : "m"(y)
  );
  printf("%d ", y_val);
}

// 代码含义如上
 __attribute__((noinline))
void write_y_read_x() {
  int x_val;
  asm volatile(
    "movl $1, %0;" // y = 1
    "movl %2, %1;" // x_val = x
    : "=m"(y), "=r"(x_val) : "m"(x)
  );
  printf("%d ", x_val);
}

void T1(int id) {
  while (1) {
    WAIT_FOR((FLAG & 1));
    write_x_read_y();
    FLAG_XOR(1);
  }
}

void T2() {
  while (1) {
    WAIT_FOR((FLAG & 2));
    write_y_read_x();
    FLAG_XOR(2);
  }
}

// thread sync func
void Tsync() {
  while (1) {
    x = y = 0;
    /*
      No memory operand will be moved across the operation,
      either forward or backward. Further, 
      instructions will be issued as necessary to prevent 
      the processor from speculating loads across 
      the operation and from queuing stores after the operation.
      简言之就是会保证 x=y=0 这条赋值操作执行完成
      或许你会问，我把这个赋值语句写在前边
    */
    __sync_synchronize(); // full barrier
    assert(FLAG == 0);
    FLAG_XOR(3); // 这里没搞懂，为什么执行 XOR 会使 cache miss
    // T1 and T2 clear 0/1-bit, respectively      
    WAIT_FOR(FLAG == 0);
    printf("\n"); fflush(stdout);
  }
}

int main() {
  // 创建线程，线程的起点为 T1
  create(T1);
  create(T2);
  create(Tsync);
}
```



上述代码执行流程分析：

- T1,T2 函数开始执行就会被卡住，因为 FLAG 变量初始值为 0
- Tsync 函数，初始化全局变量 x,y 然后执行了一个 `__sync_synchronize();` 函数，详情见注释。

然后 `FLAG_XOR` 命令，让 T1, T2 两个在等待中的开始执行。



write_y_read_x 和 write_x_read_y 两个函数，也比较简单，就是将 x y 分别进行赋值的操作，然后读取另一个变量。写 y 读 x，写 x 读 y。我们看下具体的汇编代码：

```assembly
00000000004008f1 <write_x_read_y>:
  ... # 省略无关代码
  # 将 1 赋值 x
  4008f9:       c7 05 89 17 20 00 01    movl   $0x1,0x201789(%rip)        # 60208c <x>
  # 读取 y 变量
  400903:       8b 05 87 17 20 00       mov    0x201787(%rip),%eax        # 602090 <y>
  ...
  400921:       c3                      retq
```



 从程序就是状态机的视角去分析的话，如下：

在开始时，我们有 x,y两个全局变量，T1,T2 两个线程，然后 T1，T2 开始交替执行。

![28d0652f7b48792aeea6c52883045fe.jpg](https://s2.loli.net/2022/04/19/x6KeQ2mMXN9qOJ3.jpg)

从上图我们不难得出，所有的输出结果无外乎是

```c
// 0 1
// 1 0
// 1 1
```



我们看一下执行 1000 次的输出结果是什么样

```c
 //   756 0 0 
 //   242 0 1
 //     2 1 0
```

很奇怪，根本不存在序列为 `1 1` 的组合，而且输出占最多次数的是  `0 0` ，从我们刚才列出的输出结果中，并没有这个序列。



### 解答

实际上，我们的代码经过编译器优化后生成汇编指令，汇编指令被放到处理器上进行执行。在多核的情况下，为了榨取 CPU 的最高性能，`处理器会对待执行的汇编代码进行优化` 也就是将汇编代码再执行一次编译优化的过程，即处理器也是编译器。

```assembly
movl   $0x1,0x201789(%rip)
mov    0x201787(%rip),%eax
```

从我们人类的视角出发，我们更偏向于顺序化执行，但如果从处理器的视角出发，会对这两条指令进行优化。



他做的优化是：将写操作放入到待执行队列中，如果说队列塞满了，批量进行写操作。所以，第一条 mov 指令会被放入到队列中，然后先进行读操作。

![mem-tso.png](https://s2.loli.net/2022/04/19/X5LUKbMhiE9JpCt.png)

所以这就是为什么我们没有看到 1 1 的原因。



### 如何进行修复

既然找到了问题的源头，那么就“有从下手了”，我们不让他放入到队列中，直接执行即可。



### Go 中的内存模型是怎样的？

// TODO

