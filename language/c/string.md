---
title: C-String
comment: true
---

复习 C 语言字符串

<!--more-->

# 定义

**字符串 （character string）** 是一个或多个字符的序列。



实际上，字符串就是一个字符数组，**但是在字符末尾插入了 \0 **用来表示字符串的结束。



所以，数组的容量一定是要比待存储的字符串字节数多 1，用来存储 \0，举个例子，我们想存储名字 `abc ` 就必须使用 `char name[4]` 来存储。

> 末尾的 \0 计算机会自动帮我们加上



# 使用

接收控制台输入，parisel.c

```c
#include <stdio.h>

#define PRAISE "good jobbbb"

int main(void) {
    char name[40];

    printf("Input you name ");
    scanf("%s", name);

    printf("%s, %s\n", name, PRAISE);
    return 0;
}
```



问题：

- 为什么 scanf 接收参数的时候不需要传递指针？



# 字符串和字符

```c
// 字符串 "x"
// 字符   'x'
```

这两个虽然看起来相似，但是底层的存储结构是不一样的，字符使用 char 类型来存储，字符串使用的是数组，而且还需要一个 `\0`，由两个字符组成。



# 常量

为什么使用符号常量更好？

- *常量名称可以表达更多的意思*
- *方便后续修改*



## 定义常量

`#define PI 3.14` 更通用的表示为 `#define NAME value`, C 中一般采用大写名称来表示常量。



代码中使用这样的常量，会在编译时替换掉（预处理阶段替换），即*编译时替换*（compile-time substitution）。



## const 限定符号

Go 中使用的方式 `const name = "xx"`

C 中要这样用 `const int Months = 12;`

> C 中 const 声明的是变量，不是常量。



## 看几个标准库中的常量



