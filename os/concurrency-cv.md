---
title: 并发控制-Condition-variables
comment: true
---

检查某些**特定条件**是否满足，然后决定线程是否继续执行。

<!--more-->

使用**变量**控制具体该哪个线程执行。

使用 CVs 实现 生产者/消费者 模型，最终实现：

```c
int buffer[MAX];
int fill_ptr = 0;
int use_ptr = 0;
int count = 0;
void put(int value) {
    buffer[fill_ptr] = value;
    fill_ptr = (fill_ptr + 1) % MAX;
    count++;
}

int get() {
    int tmp = buffer[use_ptr];
    use_ptr = (use_ptr + 1) % MAX;
    count--;
    return tmp;
}

cond_t empty, fill;
mutex_t mutex;

void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex); // p1
        while (count == MAX) // p2
            Pthread_cond_wait(&empty, &mutex); // p3
        put(i); // p4
        Pthread_cond_signal(&fill); // p5
        Pthread_mutex_unlock(&mutex); // p6
    }
}

void *consumer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex); // c1
        while (count == 0) // c2
            Pthread_cond_wait(&fill, &mutex); // c3
        int tmp = get(); // c4
        Pthread_cond_signal(&empty); // c5
        Pthread_mutex_unlock(&mutex); // c6
        printf("%d\n", tmp);
    }
}
```



为什么判断 `count == MAX OR count == 0` 的时候使用 `while` 而不是 `if`？？

> 结论不重要，重要的是分析的过程

// TODO 需要画状态机执行流程分析



> `Pthread_cond_wait` 主要职责：将当前线程休眠**并释放锁**，当被唤醒的时候会尝试**重新获取锁**。
