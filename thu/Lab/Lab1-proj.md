# Project Composition

[TOC]

## 背景知识

> 1. 对于Intel 80386的体系结构而言，PC机中的系统初始化软件由BIOS (Basic Input Output System，即基本输入/输出系统，其本质是一个固化在主板Flash/CMOS上的软件)和位于软盘/硬盘引导扇区中的OS Boot Loader（在ucore中的bootasm.S和bootmain.c）一起组成。BIOS实际上是被固化在计算机ROM（只读存储器）芯片上的一个特殊的软件，为上层软件提供最底层的、最直接的硬件控制与支持。更形象地说，BIOS就是PC计算机硬件与上层软件程序之间的一个"桥梁"，负责访问和控制硬件。
>
> 2. 在0xFFFFFFF0这里只是存放了一条跳转指令，通过跳转指令跳到BIOS例行程序起始点。
>
> 3. BIOS做完计算机硬件自检和初始化后，会选择一个启动设备（例如软盘、硬盘、光盘等），并且读取该设备的第一扇区(即主引导扇区`MBR`或启动扇区)到内存一个特定的地址`0x7c00`处，然后`CPU控制权`会转移到那个地址继续执行。至此`BIOS`的初始化工作做完了，进一步的工作交给了`ucore`的`bootloader`。
>
> 4. bootloader启动过程
>
>    BIOS将通过读取硬盘主引导扇区到内存，并转跳到对应内存中的位置执行bootloader。bootloader完成的工作包括：
>
>    - 切换到保护模式，启用分段机制
>    - 读磁盘中ELF执行文件格式的ucore操作系统到内存
>    - 显示字符串信息
>    - 把控制权交给ucore操作系统
>
>    对应其工作的实现文件在lab1中的boot目录下的三个文件asm.h、bootasm.S和bootmain.c。
>
> 5. 实模式和保护模式
>
>    1. **实模式**将整个物理内存看成分段的区域（但不是分段机制），程序代码和数据位于不同区域，操作系统和用户程序并没有区别对待，而且每一个指针都是指向实际的物理地址。
>    2. 只有在**保护模式**下，80386的全部32根地址线有效，可寻址高达4G字节的线性地址空间和物理地址空间，可访问64TB（有2^14个段，每个段最大空间为2^32字节）的逻辑地址空间，可采用分段存储管理机制和分页存储管理机制。这不仅为存储共享和保护提供了硬件支持，而且为实现虚拟存储提供了硬件支持。通过提供4个特权级和完善的特权检查机制，既能实现资源共享又能保证代码数据的安全及任务的隔离。
>    3. 保护模式下的两个段表：GDT（Global Descriptor Table）和LDT（Local Descriptor Table）
>    4. 在ucore lab中只用到了GDT，没有用LDT。
>
> 6. 分段存储管理机制：
>
>    1. 只有在保护模式下才能使用分段存储管理机制。
>
>    2. 分段机制将内存划分成以`起始地址`和`长度限制`这两个二维参数表示的`内存块`，这些内存块就称之为段（Segment）
>
>    3. 分段内容
>
>       1. 逻辑地址
>       2. 段描述符：描述段的属性
>       3. 段描述符表：包含多个段描述符的“数组”
>       4. 段选择子：段寄存器，用于定位段描述符表中表项的索引
>
>    4. `logical address` ---> `physical address`
>
>       1. 分段地址转换：CPU把逻辑地址（由段选择子`selector`和段偏移`offset`组成，如）中的段选择子的内容作为段描述符表的索引，找到表中对应的段描述符，然后把段描述符中保存的`段基址`加上`段偏移值`（如`CS:IP`），形成线性地址（Linear Address）。
>       2. 分页地址转换：如果不启动分页存储管理机制，则线性地址等于物理地址。 
>
>    5. 段描述符
>
>       在分段存储管理机制的保护模式下，每个段由如下三个参数进行定义：段基地址`Base Address`、段界限`Limit`和段属性`Attributes`。在ucore中的`kern/mm/mmu.h`中的`struct segdesc` 数据结构中有具体的定义。
>
>    6. GDT
>
>       * 全局描述符表的是一个保存多个段描述符的“数组”，其**起始地址**保存在全局描述符表寄存器`GDTR`中。
>       * GDTR长48位，其中高32位为基地址，低16位为段界限。
>
>       * 由于GDT 不能有GDT本身之内的描述符进行描述定义，所以处理器采用GDTR为GDT这一特殊的系统段。
>       * 对于含有N个描述符的描述符表的段界限通常可设为8*N-1。
>       * 在ucore中的`boot/bootasm.S`中的gdt地址处和`kern/mm/pmm.c`中的全局变量数组gdt[]分别有基于汇编语言和C语言的全局描述符表的具体实现。
>
>    7. 选择子
>
>       * 线性地址部分的选择子是用来选择哪个描述符表和在该表中索引一个描述符的。
>       * 结构
>         * 索引（Index）：在描述符表中从8192个描述符中选择一个描述符。处理器自动将这个索引值乘以8（描述符的长度），再加上描述符表的基址来索引描述符表，从而选出一个合适的描述符。
>         * 表指示位（Table Indicator，TI）：选择应该访问哪一个描述符表。0代表应该访问全局描述符表（GDT），1代表应该访问局部描述符表（LDT）。
>         * 请求特权级（Requested Privilege Level，RPL）：保护机制，在后续试验中会进一步讲解。
>
> 7. 保护模式下的特权级
>
>    1. 在保护模式下，特权级总共有4个，编号从0（最高特权）到3（最低特权）。
>
>    2. 有3种主要的资源受到保护：
>
>       * 内存，
>       * I/O端口
>       * 执行特殊机器指令的能力。
>
>    3. 在ucore中，CPU只用到其中的2个特权级：0（内核态）和3（用户态）。
>
>    4. 有大约15条机器指令被CPU限制只能在内核态执行，这些机器指令如果被用户模式的程序所使用，就会颠覆保护模式的保护机制并引起混乱，所以它们被保留给操作系统内核使用。（如果企图在ring 0以外运行这些指令，就会导致一个一般保护异常`general-protection exception`）
>
>    5. CS和DS结构图
>
>       ![DS_and_CS_structure_graph](D:\github\OS\thu\images\DS_and_CS_structure_graph.png)
>
>       代码段寄存器中的CPL`Current Privilege Level`（当前特权级）字段（2位）的值总是等于CPU的当前特权级，所以只要看一眼CS中的CPL，你就可以知道此刻的特权级了。
>
> 8. 关于特权级的描述
>
>    > 值越大权限级越低
>
>    - `CPL`：当前特权级（Current Privilege Level) 保存在`CS段寄存器`（选择子）的最低两位，CPL就是`当前**活动**代码段的特权级`，并且它定义了`当前所执行程序`的特权级别
>    - `DPL`：描述符特权（Descriptor Privilege Level） 存储在`段描述符`中的权限位，用于描述对应段所属的特权等级，也就是`段本身真正的特权级`。
>    - `RPL`：请求特权级RPL(Request Privilege Level) RPL保存在`选择子`的最低两位。
>      - RPL说明的是`进程对段访问`的请求权限，意思是当前进程想要的请求权限。
>      - RPL的值由程序员自己来自由的设置
>      - `DPL`即段真正的特权级要低于`CPL`和`RPL`即当前特权级与请求特权级中最低的特权级
>      - 并不一定RPL>=CPL，**但是当RPL<CPL时，实际起作用的就是CPL了**，因为访问时的特权检查是判断：max(RPL,CPL)<=DPL是否成立，所以RPL可以看成是每次访问时的附加限制，RPL=0时附加限制最小，RPL=3时附加限制最大。



## 目录结构

lab1的整体目录结构如下所示：

```
.
├── boot
│   ├── asm.h
│   ├── bootasm.S
│   └── bootmain.c
├── kern
│   ├── debug
│   │   ├── assert.h
│   │   ├── kdebug.c
│   │   ├── kdebug.h
│   │   ├── kmonitor.c
│   │   ├── kmonitor.h
│   │   ├── panic.c
│   │   └── stab.h
│   ├── driver
│   │   ├── clock.c
│   │   ├── clock.h
│   │   ├── console.c
│   │   ├── console.h
│   │   ├── intr.c
│   │   ├── intr.h
│   │   ├── kbdreg.h
│   │   ├── picirq.c
│   │   └── picirq.h
│   ├── init
│   │   └── init.c
│   ├── libs
│   │   ├── readline.c
│   │   └── stdio.c
│   ├── mm
│   │   ├── memlayout.h
│   │   ├── mmu.h
│   │   ├── pmm.c
│   │   └── pmm.h
│   └── trap
│       ├── trap.c
│       ├── trapentry.S
│       ├── trap.h
│       └── vectors.S
├── libs
│   ├── defs.h
│   ├── elf.h
│   ├── error.h
│   ├── printfmt.c
│   ├── stdarg.h
│   ├── stdio.h
│   ├── string.c
│   ├── string.h
│   └── x86.h
├── Makefile
└── tools
    ├── function.mk
    ├── gdbinit
    ├── grade.sh
    ├── kernel.ld
    ├── sign.c
    └── vector.c

10 directories, 48 files
```

## 文件说明

其中一些比较重要的文件说明如下：

***bootloader部分\***

- boot/bootasm.S ：定义并实现了bootloader最先执行的函数start，此函数进行了一定的初始化，完成了从实模式到保护模式的转换，并调用bootmain.c中的bootmain函数。
- boot/bootmain.c：定义并实现了bootmain函数实现了通过屏幕、串口和并口显示字符串。bootmain函数加载ucore操作系统到内存，然后跳转到ucore的入口处执行。
- boot/asm.h：是bootasm.S汇编文件所需要的头文件，主要是一些与X86保护模式的段访问方式相关的宏定义。

***ucore操作系统部分\***

系统初始化部分：

- kern/init/init.c：ucore操作系统的初始化启动代码

内存管理部分：

- kern/mm/memlayout.h：ucore操作系统有关段管理（段描述符编号、段号等）的一些宏定义
- kern/mm/mmu.h：ucore操作系统有关X86 MMU等硬件相关的定义，包括EFLAGS寄存器中各位的含义，应用/系统段类型，中断门描述符定义，段描述符定义，任务状态段定义，NULL段声明的宏SEG_NULL, 特定段声明的宏SEG，设置中 断门描述符的宏SETGATE（在练习6中会用到）
- kern/mm/pmm.[ch]：设定了ucore操作系统在段机制中要用到的全局变量：任务状态段ts，全局描述符表 gdt[]，加载全局描述符表寄存器的函数lgdt，临时的内核栈stack0；以及对全局描述符表和任务状态段的初始化函数gdt_init

外设驱动部分：

- kern/driver/intr.[ch]：实现了通过设置CPU的eflags来屏蔽和使能中断的函数；
- kern/driver/picirq.[ch]：实现了对中断控制器8259A的初始化和使能操作；
- kern/driver/clock.[ch]：实现了对时钟控制器8253的初始化操作；- kern/driver/console.[ch]：实现了对串口和键盘的中断方式的处理操作；

中断处理部分：

- kern/trap/vectors.S：包括256个中断服务例程的入口地址和第一步初步处理实现。注意，此文件是由tools/vector.c在编译ucore期间动态生成的；
- kern/trap/trapentry.S：紧接着第一步初步处理后，进一步完成第二步初步处理；并且有恢复中断上下文的处理，即中断处理完毕后的返回准备工作；
- kern/trap/trap.[ch]：紧接着第二步初步处理后，继续完成具体的各种中断处理操作；

内核调试部分：

- kern/debug/kdebug.[ch]：提供源码和二进制对应关系的查询功能，用于显示调用栈关系。其中补全print_stackframe函数是需要完成的练习。其他实现部分不必深究。
- kern/debug/kmonitor.[ch]：实现提供动态分析命令的kernel monitor，便于在ucore出现bug或问题后，能够进入kernel monitor中，查看当前调用关系。实现部分不必深究。
- kern/debug/panic.c | assert.h：提供了panic函数和assert宏，便于在发现错误后，调用kernel monitor。大家可在编程实验中充分利用assert宏和panic函数，提高查找错误的效率。

***公共库部分\***

- libs/defs.h：包含一些无符号整型的缩写定义。
- Libs/x86.h：一些用GNU C嵌入式汇编实现的C函数（由于使用了inline关键字，所以可以理解为宏）。

***工具部分\***

- Makefile和function.mk：指导make完成整个软件项目的编译，清除等工作。
- sign.c：一个C语言小程序，是辅助工具，用于生成一个符合规范的硬盘主引导扇区。
- tools/vector.c：生成vectors.S，此文件包含了中断向量处理的统一实现。

## 编译方法

首先下载lab1.tar.bz2，然后解压lab1.tar.bz2。在lab1目录下执行make，可以生成ucore.img（生成于bin目录下）。ucore.img是一个包含了bootloader或OS的硬盘镜像，通过执行如下命令可在硬件虚拟环境 qemu中运行bootloader或OS：

```
    $ make qemu
```

则可以得到如下显示界面（仅供参考）

```
 (THU.CST) os is loading ...

Special kernel symbols:
  entry  0x00100000 (phys)
  etext  0x00103468 (phys)
  edata  0x0010ea18 (phys)
  end    0x0010fd80 (phys)
Kernel executable memory footprint: 64KB
ebp:0x00007b38 eip:0x00100a55 args:0x00010094 0x00010094 0x00007b68 0x00100084 
    kern/debug/kdebug.c:305: print_stackframe+21
ebp:0x00007b48 eip:0x00100d3a args:0x00000000 0x00000000 0x00000000 0x00007bb8 
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b68 eip:0x00100084 args:0x00000000 0x00007b90 0xffff0000 0x00007b94 
    kern/init/init.c:48: grade_backtrace2+19
ebp:0x00007b88 eip:0x001000a5 args:0x00000000 0xffff0000 0x00007bb4 0x00000029 
    kern/init/init.c:53: grade_backtrace1+27
ebp:0x00007ba8 eip:0x001000c1 args:0x00000000 0x00100000 0xffff0000 0x00100043 
    kern/init/init.c:58: grade_backtrace0+19
ebp:0x00007bc8 eip:0x001000e1 args:0x00000000 0x00000000 0x00000000 0x00103480 
    kern/init/init.c:63: grade_backtrace+26
ebp:0x00007be8 eip:0x00100050 args:0x00000000 0x00000000 0x00000000 0x00007c4f 
    kern/init/init.c:28: kern_init+79
ebp:0x00007bf8 eip:0x00007d61 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d60 --
++ setup timer interrupts
0: @ring 0
0:  cs = 8
0:  ds = 10
0:  es = 10
0:  ss = 10
+++ switch to  user  mode +++
1: @ring 3
1:  cs = 1b
1:  ds = 23
1:  es = 23
1:  ss = 23
+++ switch to kernel mode +++
2: @ring 0
2:  cs = 8
2:  ds = 10
2:  es = 10
2:  ss = 10
100 ticks
100 ticks
100 ticks
100 ticks
```