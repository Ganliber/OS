# Lab 1-2

> 由于一个文件写到最后太卡了（`Typora`rnm退钱)，就再开一个

[TOC]

## 练习 3

> BIOS将通过读取硬盘主引导扇区到内存，并转跳到对应内存中的位置执行bootloader。请分析bootloader是如何完成从实模式进入保护模式的。
>
> 提示：需要阅读**小节“保护模式和分段机制”**和lab1/boot/bootasm.S源码，了解如何从实模式切换到保护模式，需要了解：
>
> - 为何开启`A20`，以及如何开启`A20`
> - 如何初始化`GDT`表
> - 如何使能和进入保护模式

### Q1

#### 为何开启 A20

1. 兼容原因，早期 8086 CPU 只支持 20 位寻址 (1MB) ，而“段：偏移”寻址方式支持略高于 20 位的寻址能力，大于等于部分会“回卷”而访问低地址。在新的 24 位总线 CPU 出现之后，为了兼容回卷特性而加入了 A20 地址总线，若关闭则保留回卷特性，否则禁用回卷，允许访问高地址。

2. 由于 A20 的实现是通过逻辑与进行，如果不打开则只能访问奇数位内存，为了充分利用内存当然需要打开。
3. 一开始时A20地址线控制是被屏蔽的（总为0） ，直到系统软件通过一定的IO操作去打开它（参看bootasm.S）。很显然，在实模式下要访问高端内存区，这个开关必须打开，在保护模式下，由于使用32位地址线，如果A20恒等于0，那么系统只能访问奇数兆的内存，即只能访问0--1M、2-3M、4-5M......，这样无法有效访问所有可用内存。所以在保护模式下，这个开关也必须打开。
4. 在引导系统时，[BIOS](https://zh.wikipedia.org/wiki/BIOS)先打开A20总线来统计和测试所有的系统内存。而当BIOS准备将计算机的控制权交给操作系统时会先将A20总线关闭。一开始，这个[逻辑门](https://zh.wikipedia.org/wiki/逻辑门)连接到[Intel 8042](https://zh.wikipedia.org/w/index.php?title=Intel_8042&action=edit&redlink=1)的键盘控制器。控制它是相对较慢。
5. 激活A20总线是启动[操作系统](https://zh.wikipedia.org/wiki/作業系統)的步骤之一，通常在[启动程式](https://zh.wikipedia.org/wiki/啟動程式)将控制权交给[内核](https://zh.wikipedia.org/wiki/内核)之前完成。



#### 如何开启 A20

查看`bootasm.S`源码

首先是一段使能A20的注释

```
     25     # Enable A20:
     26     #  For backwards compatibility with the earliest PCs, physical
     27     #  address line 20 is tied low, so that addresses higher than
     28     #  1MB wrap around to zero by default. This code undoes this.
```

然后是第一段`seta20.1`

```assembly
     29 seta20.1:
     30     inb $0x64, %al                           # Wait for not busy(8042 input buffer empty).
     31     testb $0x2, %al
     32     jnz seta20.1
     33 
     34     movb $0xd1, %al                          # 0xd1 -> port 0x64
     35     outb %al, $0x64                          # 0xd1 means: write data to 8042's P2 port
```

第二段`seta20.2`

```assembly
     37 seta20.2:
     38     inb $0x64, %al             # Wait for not busy(8042 input buffer empty).
     39     testb $0x2, %al
     40     jnz seta20.2
     41 
     42     movb $0xdf, %al            # 0xdf -> port 0x60
     43     outb %al, $0x60            # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```





## 练习 4

> 分析bootloader加载ELF格式的OS的过程。
>
> 通过阅读bootmain.c，了解bootloader如何加载ELF文件。通过分析源代码和通过qemu来运行并调试bootloader&OS，
>
> - bootloader如何读取硬盘扇区的？
> - bootloader是如何加载ELF格式的OS？
>
> 提示：可阅读“硬盘访问概述”，“ELF执行文件格式概述”这两小节。



## 练习 5

> 实现函数调用堆栈跟踪函数 （需要编程）
>
> 我们需要在lab1中完成kdebug.c中函数print_stackframe的实现，可以通过函数print_stackframe来跟踪函数调用堆栈中记录的返回地址。在如果能够正确实现此函数，可在lab1中执行 “make qemu”后，在qemu模拟器中得到类似如下的输出：
>
> ```
> ……
> ebp:0x00007b28 eip:0x00100992 args:0x00010094 0x00010094 0x00007b58 0x00100096
>     kern/debug/kdebug.c:305: print_stackframe+22
> ebp:0x00007b38 eip:0x00100c79 args:0x00000000 0x00000000 0x00000000 0x00007ba8
>     kern/debug/kmonitor.c:125: mon_backtrace+10
> ebp:0x00007b58 eip:0x00100096 args:0x00000000 0x00007b80 0xffff0000 0x00007b84
>     kern/init/init.c:48: grade_backtrace2+33
> ebp:0x00007b78 eip:0x001000bf args:0x00000000 0xffff0000 0x00007ba4 0x00000029
>     kern/init/init.c:53: grade_backtrace1+38
> ebp:0x00007b98 eip:0x001000dd args:0x00000000 0x00100000 0xffff0000 0x0000001d
>     kern/init/init.c:58: grade_backtrace0+23
> ebp:0x00007bb8 eip:0x00100102 args:0x0010353c 0x00103520 0x00001308 0x00000000
>     kern/init/init.c:63: grade_backtrace+34
> ebp:0x00007be8 eip:0x00100059 args:0x00000000 0x00000000 0x00000000 0x00007c53
>     kern/init/init.c:28: kern_init+88
> ebp:0x00007bf8 eip:0x00007d73 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
> <unknow>: -- 0x00007d72 –
> ……
> ```
>
> 请完成实验，看看输出是否与上述显示大致一致，并解释最后一行各个数值的含义。
>
> 提示：可阅读小节“函数堆栈”，了解编译器如何建立函数调用关系的。在完成lab1编译后，查看lab1/obj/bootblock.asm，了解bootloader源码与机器码的语句和地址等的对应关系；查看lab1/obj/kernel.asm，了解 ucore OS源码与机器码的语句和地址等的对应关系。
>
> 要求完成函数kern/debug/kdebug.c::print_stackframe的实现，提交改进后源代码包（可以编译执行），并在实验报告中简要说明实现过程，并写出对上述问题的回答。
>
> 补充材料：
>
> 由于显示完整的栈结构需要解析内核文件中的调试符号，较为复杂和繁琐。代码中有一些辅助函数可以使用。例如可以通过调用print_debuginfo函数完成查找对应函数名并打印至屏幕的功能。具体可以参见kdebug.c代码中的注释。







## 练习 6

> 完善中断初始化和处理 （需要编程）
>
> 请完成编码工作和回答如下问题：
>
> 1. 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？
> 2. 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。
> 3. 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”。
>
> > 【注意】除了系统调用中断(T_SYSCALL)使用陷阱门描述符且权限为用户态权限以外，其它中断均使用特权级(DPL)为０的中断门描述符，权限为内核态权限；而ucore的应用程序处于特权级３，需要采用｀int 0x80`指令操作（这种方式称为软中断，软件中断，Tra中断，在lab5会碰到）来发出系统调用请求，并要能实现从特权级３到特权级０的转换，所以系统调用中断(T_SYSCALL)所对应的中断门描述符中的特权级（DPL）需要设置为３。
>
> 要求完成问题2和问题3 提出的相关函数实现，提交改进后的源代码包（可以编译执行），并在实验报告中简要说明实现过程，并写出对问题1的回答。完成这问题2和3要求的部分代码后，运行整个系统，可以看到大约每1秒会输出一次”100 ticks”，而按下的键也会在屏幕上显示。
>
> 提示：可阅读小节“中断与异常”。





















