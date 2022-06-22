---
title: 同步原语-semaphore
comment: true
---

同步原语--信号量学习

<!--more-->

## 信号量定义

信号量就是一个拥有整数值的对象，可以用多个例程操控它。

> 信号量的初始值决定了它的行为，在与它交互之前需要进行初始化，如下：

```c
#include <semaphore.h>

# int sem_init(
sem_t *sem,
int pshared,
unsigned int value);
# sem 信号
# value 指定了信号量的初始值
# pshared 表示在线程间共享，还是进程间
sema_init()
```

## 使用 sema 实现 lock

这时应将 sema 的 value 设置为几？

可以想象，锁的状态其实只有**上锁，未上锁**两种状态，所以我们只需要使信号量满足这两种状态即可，即 value = 1，这种情况下，0，1就可以表示锁的这两种状态。



想进入临界区（以下用 CS 代替），就需要获取锁，检查信号量是否被占用：

- 未上锁，信号量的值减一，进入临界区
- 上锁，信号量值减一，睡眠等待



退出 CS，调用 `sema_post` 释放信号：

- 有等待线程：信号量值加一，退出临界区，唤醒一个等待中的线程
- 没有等待线程：信号量值加一，退出临界区，无需唤醒



> 当信号量为负数时，比如说 -3，代表着有三个线程在等待这个信号被释放



不难看出，当我们使用信号量来实现锁的时候，只有这两种状态（上锁，未上锁），所以这种信号通常也被称为**binary semaphore**



## sema 实现 CVs

`sema_init(sema, 0, 0)`

想要父进程等待子进程执行结束的话，为什么 value 的值要设置为 0 ？



## sema 实现 producer/consumer 模型

如同当初使用 CVs 实现生产者消费者模型时一样，这里也是需要两个信号量来表示何时生产者可发送，接收者可接收。

```c
int main(int argc, char *argv[]) {
    // ...
    sem_init(&empty, 0, MAX); // MAX are empty
    sem_init(&full, 0, 0); // 0 are full
    // ...
}
```



当 MAX > 1 时，会出现什么问题？（仔细观察，的确不难看出，data-race）

```c
int buffer[MAX];
int fill = 0;
int use = 0;
void put(int value) {
    buffer[fill] = value; // Line F1
    fill = (fill + 1) % MAX; // Line F2
} 
int get() {
    int tmp = buffer[use]; // Line G1
    use = (use + 1) % MAX; // Line G2
    return tmp;
}

sem_t empty;
sem_t full;
void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        sem_wait(&empty); // Line P1
        put(i); // Line P2
        sem_post(&full); // Line P3
    }
}

void *consumer(void *arg) {
    int tmp = 0;
    while (tmp != -1) {
        sem_wait(&full); // Line C1
        tmp = get(); // Line C2
        sem_post(&empty); // Line C3
        printf("%d\n", tmp);
    }
}
```



## 哲学家吃饭..

![image-20220427140847674](https://s2.loli.net/2022/04/27/wYIi1fhCQog2XVP.png)



- **如何解决相互依赖的问题**

- **什么情况下会出现死锁**



## Thread Throttling

线程节流，控制线程数量。

通过信号量控制进入临界区的线程数量