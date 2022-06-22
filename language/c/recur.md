---
title: 再学递归（几个有意思的递归练习）
comment: true
---

逃不过的汉诺塔...

<!--more-->

由简到难：

- 给定一个整数，如 1234，使用递归的方式打印出，4321。
- 给定一个整数，如 9，使用递归的方式打印出该数的二进制形式，输入 9，输出 1001。

这两个数以同一种类型，做出来一个另一个也迎刃而解，如果没什么思路的话，也不建议直接去查答案，肯定可以，我这么笨的都能琢磨出来，多花点时间。

（提示请见评论）

- 给定一个整数，求这个数的阶乘。

- 汉诺塔..只要是认认真真学过一门语言肯定会接触到这个例子，具体描述请见百科。

``` c
#include <stdio.h>

void hanoi(int, char, char, char);

int main(void) {
    hanoi(3, 'x', 'y', 'z');
    return 0;
}

void hanoi(int n, char from, char mid, char to) {
    if (n > 0) {
        // from x to z through y
        hanoi(n-1, from, to, mid);
        printf("take %d from %c to %c\n", n, from, to);
        // from z to x through y
        hanoi(n-1, mid, from, to);
    }
}
```

