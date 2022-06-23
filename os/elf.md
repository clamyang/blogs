---
title: elf 文件格式
---

简单整理下 ELF (executable and linkable format) 的几个知识点，这里跟 linker&&loader 中有一点重复内容。

<!--more-->

### ELF的三种格式

- 可重定位文件
  - 包含二进制代码和数据，其形式可以在编译时与其他可重定位目标文件合并起来，创建一个可执行目标文件。
- 可执行文件
  - 包含二进制代码和数据，其形式可以被直接复制到内存并执行。
- 共享对象文件
  - 一种特殊类型的可重定位目标文件，可以在加载或运行时被动态地加载进内存并链接。

#### 可重定位目标文件

<img src="https://gitee.com/yangbaoqiang/images/raw/master/blogpics/微信截图_20211209191808.png" alt="微信截图_20211209191808" style="zoom:50%;" />

##### sections

- `.text ` 存放编译好的二进制可执行代码
- `.rodata` 存放只读数据，`const` 变量等
- `.data` 存放已经初始化好的全局变量
- [.bss](https://en.wikipedia.org/wiki/.bss)  未初始化全局变量，运行时会置 0
  - 比如在全局作用域下，`var name string` 这种就是分配在 `.bss` section 中

​		注意：在目标文件中这个 section 不占实际的空间，仅仅是一个占位符。目标文件对已初始化和未初始化主要是为了空间效率

- `.symtab` 符号表，记录函数和变量
- `.strtab` 字符串表就是以 null 结尾的字符串序列



为什么未初始化的数据称为 .bss？

书中提到它起始于 IBM704 汇编语言中 “块存储” 指令的首字母缩写，因为 .bss section 是不占用存储空间的，所以一种更好的记住 .data 和 .bss 中的区别就是把 bss 看做成 “更好的节省空间” (Better Save Space) 的缩写。

##### 三个特殊的伪节

（只有可重定位目标文件中才有，可执行中没有）

- ABS 代表不该被重定位的符号
- UNDEF 代表未定义的符号，表示的是在本目标模块中引用，但是在其他地方定义的符号
- COMMON 表示还未被分配位置的未初始化的数据目标

##### COMMON 和 .bss 的区别

//TODO 

#### 可执行文件

如下图所示：

<img src="https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211206141434504.png" alt="image-20211206141434504" style="zoom:67%;" />

代码段：用来存放编译好的代码

数据段：包括已经初始化的全局变量，未初始化的全局变量

`.text` 和 `.rodata` 两个 section 组成代码段 `segment`，其他也类似。
