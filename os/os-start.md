---
title: 操作系统启动
comment: true
---

---

尝试揭开操作系统的神秘面纱。

<!--more-->

# QEMU

## 安装步骤

```
wget https://download.qemu.org/qemu-7.0.0.tar.xz
tar xvJf qemu-7.0.0.tar.xz
cd qemu-7.0.0
./configure
make
// make 后记得 make install
```

> 安装个 QEMU 真的是费死劲了，能遇到的问题基本上都遇到了，而且**这个容器我挂了windows**的一个目录（为了保存下来），make 花了**3**小时。



遇到的具体问题如下：

- 缺少依赖，pixman-1, gthread, glib, 我是基于 alpine 镜像构建的，很多依赖找起来很费事，不过要熟练使用 search 命令。



具体解决方案省略，作为一个合格的工程师，肯定会找到办法的。



# CPU Reset

程序就是状态机，操作系统也是一个 C 程序。那么问题就来了，电脑在 CPU Reset 之后（获得了一个初始状态）发生了什么？



> 初始状态指的是：各种寄存器的初始值是什么。



**计算机中没有任何神秘的东西**

![intel-cpu-reset](https://s2.loli.net/2022/05/06/e6anlsRhC2BULVu.png)



可以看到，表 9-1 列出了在通电、重置、初始化后各种寄存器的值。这些都是**约定**，即硬件和软件约定好，每次加电后，CPU 状态设置为这些值，CPU 就是不断地执行指令，然后一步一步的加载出操作系统代码。



![image-20220506202315147](https://s2.loli.net/2022/05/06/9SzlH2nRchoykQL.png)



# Legacy BIOS 约定

（操作系统与BIOS之间的约定，BIOS 上哪加载操作系统代码）

BIOS （Basic I/O System），BIOS 就是我们 CPU Reset 后 PC 指向的位置，也就意味着加电后执行的第一个程序是 BIOS 代码。然后 BIOS  代码都做了什么呢？他要做的事情都是约定好的。



MBR（Master Boot Record）主引导扇区，BIOS 要做的事情就是加载磁盘的前**512字节**（主引导扇区）到 **0x7c00**，**检查这块磁盘是否可以作为启动盘**。如果是，可以从磁盘中加载更多的内容到内存中，否则检查下一块磁盘的前512字节。



检查磁盘作为启动盘的标志是什么？**55aa**

> 怎么查看呢？有没有什么命令能查？
>
> 还真就是只有想不到的，没有做不到的。

**hexdump ** 可以 `hexdump --help` 看一下具体细节，这里就是 `hexdump -n 512 /dev/sda	`



补上一张 MBR 的图片

![o_mbr_anatomy](https://s2.loli.net/2022/05/13/K9cL6sJ2etyP4QR.png)



是哪条指令将 MBR 中的内容加载到内存中的？

```assembly
# 就是下面这条指令
0xfa759:     rep insl (%dx),%es:(%di)
# ...
```

> 如何查看上述的指令？x/i ($cs * 16 + \$rip)

cs （code segment register） 代码段寄存器

ds （data segment register） 数据段寄存器

为什么要按照这个算式进行查询呢？`($cs * 16 + $rip)` 早期的 IBM PC 机器，总线是 20 位，但是寄存器中都是 16 位的数据，所以怎么凑够这个 20 位的总线，代码段地址左移 4 位，然后加上偏移量，就是我们要执行的下一条指令地址。



# 为什么叫做 x86 架构

一般这种都是历史原因... 早期的 IBM PC 机器使用 Intel 8086 处理器，后来就把这个 8086 CPU 的架构叫做 x86。更详细的内容可以 Google 上看下，其实就连这个 `0x7c00  `都有一部分历史原因。



