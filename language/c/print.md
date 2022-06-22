---
title: 打印字符
comment: true
---

for循环打印字符

<!--more-->

还是有点意思的，使用嵌套for循环输出如下格式：

```
A
BC
DEF
GHIJK
...
```



另一个输出这样：

```
  A  
 ABA 
ABCBA
```





华

------



​					丽

------



​									的

------



​													分

------



​																	割

------



​																					线

------



答案如下：

```C
#include <stdio.h>

int main(void) {
    for (int i = 0, s = 'A'; i < 6; i++, s+=i) {  
        for(int j = s, k = 0; k <= i; k++, j++ ) {
            printf("%c", j);
        }
        printf("\n");
    }
    return 0;
}
```



```C
#include <stdio.h>

int main(void) {
    printf("Please input start letter for print\n");   
    char alpha;
    scanf("%c", &alpha);
    char A = 65;

    for (char t = A; t <= alpha; t++) {   
        for (char j = alpha; j > t; j--) {
            printf(" ");
        }
        
        for(char i = A; i < t; i++) {     
            printf("%c", i);
        }

        for (char k = t; k >= A; k--) {   
            printf("%c", k);
        }
        
        printf("\n");
    }
    return 0;
}
```

