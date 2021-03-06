---
title: linker and loader
comment: true
---

学习下 linker 和 loader 中的内容，记录一下学习过程遇到的笔记以及知识点。主要的内容来自 csapp。

<!--more-->

### 为什么要学链接

- 理解链接器能帮助我构造大型程序
- 理解链接器能帮助我避免一些危险的编程错误
- 理解链接器能帮助我理解语言中作用域规则的实现
- 理解链接器能帮助我理解其他系统概念
- 理解链接器能帮助我利用共享库

这是文中提到的一些关键点，我个人更想了解的是，linker 和 loader 的执行过程，毕竟 Go 是编译类型的语言，还是有必要去了解一下其底层的编译的内容的。

书中的例子都是 C 语言实现的，只需要你简单了解一下语法即可，很简单。



### 编译器驱动程序

一个编译的过程有以下几个步骤：（一个 gcc 可以拆成以下四步）

- 语言预处理器
- 编译器
- 汇编器
- 链接器



（这个就像 Go 语言中，一个 go build 可以拆分成 compile 和 link 的过程。）



比如要编译 main.c 和 sum.c 文件， 通过 `gcc -Og -o prog main.c sum.c` 即可输出 prog 可执行文件。也可以一步一步拆解进行，如下：

- 预处理器： `cpp main.c ./main.i`  翻译成一个 ASCII 码的中间文件
- 编译器： `cc1 ./main.i -Og -o ./main.s`  翻译成一个ASCII 码的**汇编语言文件**
- 汇编器： `as -o ./main.o ./main.s`  翻译成一个**可重定位目标文件**
- 链接器： `ld -o prog ./main.o ./sum.o` 生成一个**可重定位目标文件**

通过 gcc 的方式

- gcc -E hello.c -o hello.i
- gcc –S hello.i –o hello.s
- gcc –c hello.s –o hello.o
- gcc hello.o –o hello

### 静态链接

- 符号解析
  - 目的是将每个符号引用正好和一个符号定义关联起来，比如将符号对应到函数、全局变量等。
- 重定位
  - 把每个符号定义与一个内存位置关联起来。

### 目标文件

[点我跳转到 ELF](https://www.bqyang.top/2021/elf)

### 可重定位目标文件

[点我跳转到 ELF](https://www.bqyang.top/2021/elf)

### 符号和符号表

- 由模块 m 定义并能被其他模块引用的**全局符号**。
  - C 语言：全局链接器符号对应于非静态的C函数和全局变量。
  - Go语言：可以理解为在 util 包下定义的全局函数或全局变量。
- 由其他模块定义并被模块 m 引用的**全局符号**。这些符号称为外部符号。
  - C 语言：对应于在其他模块中定义的非静态 C 函数和全局变量。
  - Go 语言： 可以理解为在 util 包下引用了  time 包中的内容。
- 只被模块 m 定义和引用的**局部符号**。
  - C 语言：对应于带 static 属性的 C 函数和全局变量。
  - 可以理解为在 Go util 包中定义的不可导出的函数或变量。

注意：C 语言中使用 static 关键字，隐藏模块内部的变量和函数声明，就像 Java 中的 private 和 public。在 C 中，任何带有 static 属性声明的全局变量或函数都是模块私有的。类似的，任何不带 static 属性的全局变量和函数都是公共的，可以被其他模块访问。

符号表是如何记录一个函数的：



![](https://s2.loli.net/2022/06/23/D84UqxfTuAZneNS.png)



如上图中所示，一个函数在符号表中被称为条目，该条目中包括了函数的名称，函数的位置，函数是局部的还是全局的。

- name 表示字符串表中的字节偏移，指向符号的以 null 结尾的字符串名字。
  - 可以理解为函数名或者变量名字。
- value 表示符号的地址。
  - 对于可重定位的模块来说，value 是距定义目标的节的起始位置的偏移。
  - 对于可执行目标文件来说，该值是一个绝对运行时地址。
- size 表示目标的大小。
- type 表示要么是数据，要么是函数。
- binding 表示是局部的，还是全局的。

~~~c
// main.c 文件
int sum(int *a, int n);      

int array[2] = {1, 2};       

int main() {                 
    int val = sum(array, 2); 
    return val;              
}
~~~

乍一看是不是感觉很熟悉，Go 看起来也差不多的哈哈。通过上述的**预处理-编译-汇编-链接**这三步，将 main.c 文件编译成 main.o 文件。然后再查看 main.o 文件的符号表，如下图所示：

![](https://s2.loli.net/2022/06/23/SzdTlrF6DXu2aUQ.png)

可以看到我们在 main.c 中定义的全局变量 array，sum 和 main。

Ndx 表示所在的 section，.text 索引为 1，以此类推。



再来看个简单的例子，以下内容来自 csapp 的课后习题。代码如下所示，

~~~c
// swap.c
extern int buf[];

int *bufp0 = &buf[0];
int *bufp1;

void swap()
{
    int temp;

    // 以下过程就是将 buf 中的两个元素进行交换
    bufp1 = &buf[1];
    temp = *bufp0;
    *bufp0 = *bufp1;
    *bufp1 = temp;
}

// m.c
void swap();

int buf[2] = {1, 2};

int main()
{
    swap();
    return 0;
}
~~~

同样通过**预处理-编译-汇编-链接**的前三步，输出 swap.o 和 m.o 文件。按部就班的来，我们再看看 swap.o 的符号表长什么样。

![](https://s2.loli.net/2022/06/23/UQ593ENuTzKrZMk.png)

| 符号  | .symtab 条目？ | 符号类型 | 在哪个模块定义 |          节           |
| :---: | :------------: | :------: | :------------: | :-------------------: |
|  buf  |       是       |   外部   |      m.o       | 可以自行查看 m.o 确定 |
| bufp0 |       是       |   全局   |     swap.o     |         .data         |
| bufp1 |       是       |   全局   |     swap.o     |          COM          |
| swap  |       是       |   全局   |     swap.o     |         .text         |
| temp  |       否       |    -     |       -        |   都不在（栈分配）    |

- .symtab 填是否
- 符号类型填全局、局部、外部
- 在哪个模块定义，即为在哪个文件中定义的
- 节，Ndx 对应的 section

可以简单的做一下，看看你是否真的能够读懂 .symtab（符号表）。笔者一开始理解的符号表，只是其字面意思，以为就是用来存储符号的表，在网上查阅相关资料后，知道他存储的是函数和变量，在阅读 csapp 相关章节后，对于符号表的内容又有了进一步认识。

注：关于 COM、UND、ABS 的相关内容，请移步到 [ELF](https://www.bqyang.top/2021/elf) 中进行查阅。

### 符号解析

#### 链接器如何解析多重定义的全局符号

- 强符号

  函数和已初始化的全局变量是强符号。

- 弱符号

  未初始化的全局变量是弱符号。

规则如下：

1. 不允许有多个同名的强符号
2. 如果有一个强符号和多个弱符号同名，那么选择强符号
3. 如果有多个弱符号同名，那么从这些弱符号中任意选择一个

#### 与静态库链接

将所有相关的目标模块打包成为一个单独的文件，称为静态库（static library）。



在 linux 系统中，静态库以一种称为存档（archive）的特殊文件格式存放在磁盘中。存档文件是一组连接起来的可重定位目标文件的集合，有一个头部用来描述每个成员目标文件的大小和位置。archive 文件名后缀未 .a 标识。

#### 链接器如何使用静态库来解析引用

注：

​	1.在这里抛出一个问题， Go 语言中，我们在 main 包中调用了一个库函数，比如 `math.Max(1, 2)` ，请你思考一下，在程序编译过程中是把整个 math 模块都编译了进来，还是说只会将 main 包中用到的库函数编译进来？

​	2.如题，链接器如何使用静态库解析引用



在解析过程中要维护三个集合

- 一个可重定位目标文件的集合 E，这个集合中的文件会被合并起来形成可执行文件
- 一个未解析的符号集合 U
- 一个在前面输入文件中已定义的符号集合 D



![](https://s2.loli.net/2022/06/23/oTYf4JLwVx3X6KU.png)注意：如果最后集合 U 是不为空的，说明有的符号未能在可执行目标文件或者静态库中找到 `symbol not found` 。

小结：至此我们已经学习过了，符号的定义，符号的解析过程。把代码中每个符号的引用和确切的一个符号定义关联了起来。

### 重定位

重定位是由两步组成的：

- 重定位 section 和 符号定义。在这一步中，链接器将所有相同类型的节合并为同一类型的新的聚合节。比如讲所有目标文件中的 `.data` section 合并成一个 `.data` section 并且这个 section 最后就是可执行目标文件的section。
- 重定位 section 中的符号引用。在这一步中，链接器修改 `.text` 和 `.data` 中对每个符号的引用，使得它们指向正确的运行时地址。

#### 重定位条目

为什么需要重定位条目？

- 因为汇编器生成一个目标文件的时候，并不知道该文件中 `.data` 和 `.text` section 最终要在内存的什么位置，与此同时，汇编器也不知道这个模块引用的任何外部定义的函数或者全局变量的位置。
- 所以汇编器在遇到一个最终位置未知的符号引用，它就会生成一个重定位条目。告诉链接器将目标文件合并成可执行文件时如何修改这个引用。



**代码重定位条目放在 `.rel.text` 中**

**已初始化数据的重定位条目放在 `.rel.data` 中**



什么是重定位条目？

![](https://s2.loli.net/2022/06/23/uZaVLfQmcbBEP9v.png)

参数解释：

- offset 需要被修改的引用的偏移
- 表示被修改引用应该指向的符号，在 symtab 中的位置
- type 告知链接器如何修改引用
- addend 是一个有符号常数，一些类型的重定位要使用它对被修改引用的值做偏移调整。

注：我们只需要关注两个最基本的重定位条目

- R_X86_64_PC32 重定位一个使用 32 位 PC 相对地址的引用。在指令中编码的 32 位值加上 PC 的当前运

  行时值，得到有效地址。

- R_X86_64_32 重定位一个使用 32 位绝对地址的引用。



#### 重定位符号引用

1.重定位 PC 相对引用

2.重定位 PC 绝对引用	

![](https://s2.loli.net/2022/06/23/vcVxoFJTnpdyabQ.png)

这张图主要讲了，针对相对或绝对的不同类型，对应到不同的计算方式。



### 加载可执行目标文件

当我们通过命令行调用我们的可执行文件时 `./prog` shell 会认为这是一个可执行目标文件，然后通过某个在内存中称为加载器（loader） 的操作系统代码来运行 `./prog` 。



加载器将可执行目标文件中的代码和数据从磁盘加载到内存，然后通过跳转到程序的第一条指令或入口来运行该程序。程序被复制到内存并运行的过程称为**加载**。



#### 加载器实际上是如何工作的：

书中提到，这里只是一个简单的概述。

Linux 系统中的每个程序都运行在一个进程上下文中，有自己的虚拟地址空间。当 shell 运行一个程序时，父 shell 进程生成一个子进程，它是父进程的一个复制。子进程通过 execve 系统调用启动加载器。加栽器删除子进程现有的虚拟内存段，并创建一组新的代码、数据、堆和栈段。新的栈和堆段被初始化为零 通过将虚拟地址空间中的页映射到可执行文件的页大小的片 (chunk), 新的代码和数据段袚初始化为可执行文件的内容。最后，加载器跳转到 _start地址，它最终会调用应用程序的 main 函数。除了一些头部信息，在加栽过程中没有任何从磁盘到内存的数据复制 直到 CPU 引用一个被映射的虚拟页时才会进行复制，此时，操作系统利用它的页面调度机制自动将页面从磁盘传送到内存。



### 动态链接共享库



### 参考

- https://stackoverflow.com/questions/6687630/how-to-remove-unused-c-c-symbols-with-gcc-and-ld

- 简书-图解静态库链接过程-https://www.jianshu.com/p/4510b47ecd8a
- csapp 第七章

