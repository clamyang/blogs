2.多进程操作同一个文件

```c
// file.c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
    printf("i am parent, my pid is %d\n", (int)getpid());
    FILE *fd = fopen("./info.txt", "a+");
    int rc = fork();

    if (rc < 0) {
        printf("new process failed");
        exit(1);
    } else if (rc == 0) {
        for (int i = 0; i < 10; i++) {
            fputs("STFW....\n\n", fd);
        }
        printf("child write finished\n");
    } else if (rc > 0) {
        for (int i = 0; i < 10; i++) {
            fputs("hello RTFMMM\n\n", fd);
        }
        printf("parent write finished\n");
        //int data = wait(NULL);
        //printf("%d\n", data);
    }
    return 0;
}
```

正如程序中看到的，我们在两个进程中操作同一个文件描述符，讲道理这种没加约束的并发写操作都是存在问题的。但是我这里看文件中的内容有点过于正确。。



4.尝试不同的 `exec execl execle execlp execv execvp execvpe` 并思考为什么要有这么多的变种？

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
    int rc = fork();

    if (rc < 0) {
        exit(1);
    } else if (rc == 0) {
        char *myargs[3];
        myargs[0] = strdup("ls");
        myargs[1] = strdup("-la");
        myargs[2] = NULL;
        int result = execvp(myargs[0], myargs);
        // can not reach
        printf("%d\n", result);
    } else if (rc > 0) {
        int childs = wait(NULL);
        printf("%d\n", childs);
    }
    return 0;
}
```



execl

使用 execl 时，第一个和第二个参数都是可执行文件的路径，第三个参数开始就是自带的一些参数比如 `-la`，最后一个参数需要传 NULL 而且要转成 char * 类型的空指针：

```c
char *myargs[3];
myargs[0] = strdup("/bin/ls");
myargs[1] = strdup("-la");
myargs[2] = strdup("/root");
int result = execl(myargs[0], myargs[0], myargs[1], myargs[2], (char *) NULL);
```

后续就不一一举例了，具体内容可以参考 [[这里](https://linuxhint.com/exec_linux_system_call_c/)]。



5.double wait 在父进程中执行 wait 函数，在子进程中也执行 wait 函数，这时会发生什么？

没有实践时，我认为会直接卡死，产生相互等待的情况，正如 go 中 all goroutine asleep.

但是，神奇的事情又发生了，这个程序不但没卡死，还正确的执行了。

```c
// doubleWait.c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/wait.h>

int main(void) {
    int rc = fork();

    if (rc < 0) {
        exit(1);
    } else if (rc == 0) {
        printf("start exec child %d proces\n", (int)getpid());
        wait(NULL);
        printf("%d\n", 111);

    } else if (rc > 0) {
        printf("start wait child");
        int childs = wait(NULL);
        printf("%d\n", childs);
    }
}
```

很意外，当这个 `111` 打印出来后恍然大悟，我们在父进程中执行了 fork 这时候调用 wait 可以得到返回的子进程 pid。但是当你在子进程中执行 wait 时，是没有任何子进程的子进程，所以 wait 会直接返回。（还是要多动手实践呀..自己这个脑子..并不靠谱）