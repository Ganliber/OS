# Lab 1

> bootloader 启动 OS

[TOC]



## x86启动顺序

* register 初始值
  * CS = F000H，EIP = 000FFF0H
* 实际地址
  * 段寄存器CS实际基址 Base = FFFF0000H
  * Base + EIP = FFFFFFF0H --> 这是BIOS的EPROM `Erasable Programmable Read Only Memory` 所在地（只读）
  * 当CS被新值加载，地址转换规则将开始起作用
  * 第一条指令是一个`长跳转指令(这样CS,EIP均会更新)`，跳转到`BIOS`代码中执行
  
* 处于实模式的段

  * 实模式寻址：CS:IP --> FFFF0H (20-bit（1MiB)）
  * 段选择子 `Segment Selector` : CS, DS, SS, ...
  * 偏移量 `Offset` ：EIP

* 从`BIOS`到`Bootloader`

  * `BIOS` 加载存储设备（软盘，硬盘，光盘，USB盘）上的第一个扇区（主引导扇区`MBR(Master Boot Record)`）的512字节到内存的`0x7c00`

  * 跳转到 `0x7c00` 的第一条指令开始执行

* 从`bootloader`到`OS`

  * 使能`enable`保护模式 `protection mode` & 段机制 `segment-level`
  * 从硬盘上读取`kernel` in ELF格式的`ucore kernel`（跟在`MBR`后的扇区）并放到内存中固定位置
  * 跳转到 `ucore OS`的入口点 `entry point` 执行，此时控制权到了`ucore OS`

* x86启动顺序 - 段机制

  * 一种简单直接的映射关系（后来有页机制）
  * 先建立好全局描述符表 `GDT`, 由`bootloader`建立
    * 每一项是一个段描述符
    * 设置各个段寄存器的`index`, 使`index`指向`GDT`对应的项
  * 特定寄存器：系统寄存器 `CRO`
    * Enable protection mode : bootloader / OS 要设置 CR0 的bit 0(PE)
    * Segment-level protection : 在保护模式下是自动使能的 

* 加载 ELF 格式的 `ucore OS kernel`
  * 解析ELF文件格式：
    * ELF header : 
      * uint phoff : Program Header Offset
      * ushort phnum : Program Header Number  
    * Program Header : 
      * uint offset : 起始地址，beginning of the segment in the file
      * uint va : 虚地址，where this segment should be placed at
      * uint memsz : 代码块大小，size of segment in byte



## C函数调用的实现

> x86-32 汇编



## GCC 内联汇编

> inline assembly instructions

### Grammar

#### 基本内联汇编



#### 扩展内联汇编

##### `Grammar` : 

```C
Grammar 1:
asm(assembler template
  :output operands(optional)
  :input operands(optional)
  :clobbers(optional)   --- 约束规则
);

Grammar 2:
asm [volatile] ( Assembler Template
   : Output Operands
   [ : Input Operands
   [ : Clobbers ] ]);

Grammar 3:
__asm__ __volatile__(...);
```

##### `Tips` : 

* "asm" 和 "\__asm__" 的含义是完全一样的,如果有多行汇编，则每一行都要加上 "\n\t"。其中的 “\n” 是换行符，"\t” 是 tab 符，在每条命令的 结束加这两个符号，是为了让 gcc 把内联汇编代码翻译成一般的汇编代码时能够保证换行和留有一定的空格。
* 数字前加前缀 “％“，如％1，％2等表示使用寄存器的样板操作数。
* 由于这些样板操作数的前缀使用了”％“，因此，在用到具体的寄存器时就在前面加**两个“％”**，如**%%cr0**，但基本的内联汇编寄存器前只需要加一个%即可

* 示例

  ```C
  #define read_cr0() ({ \
      unsigned int __dummy; \
      __asm__( \
          "movl %%cr0,%0\n\t" \
          :"=r" (__dummy)); \
      __dummy; \
  })
  ```

##### `Output operand list` :

> 1. 输出部分`output operand list`，用以规定对输出变量（目标操作数）如何与寄存器结合的约束（constraint）,输出部分可以有多个约束，**互相以逗号分开**。**每个约束以“＝”开头**，接着用一个字母来表示操作数的类型，然后是关于变量结合的约束。

关于约束

```assembly
:"=r" (__dummy) 
+ --- 表任意寄存器

:"=m" (__dummy)  
+ --- 表表示相应的目标操作数是存放在内存单元__dummy中
```

主要约束字母及其含义

| 字母       | 含义                                             |
| ---------- | ------------------------------------------------ |
| m, v, o    | 内存单元                                         |
| R          | 任何通用寄存器                                   |
| Q          | 寄存器eax, ebx, ecx,edx之一                      |
| I, h       | 直接操作数                                       |
| E, F       | 浮点数                                           |
| G          | 任意                                             |
| a, b, c, d | 寄存器eax/ax/al, ebx/bx/bl, ecx/cx/cl或edx/dx/dl |
| S, D       | 寄存器esi或edi                                   |
| I          | 常数（0～31）                                    |

##### `Input operand list` :

>1. 输入部分与输出部分相似，但没有“＝”。



##### `clobber list` :

> 1. 也称为乱码列表



### Example 1

Inline assembly (*.c)

> 对系统寄存器 cr0 进行权限位（相应一个bit）置位,将该位置为1（见`cr0 |= 0x80000000`一行采取了或运算）
>
> 注释 1：
>
> [2] : 将 register CR0 的值赋给 [1] 声明的变量 cr0
>
> [3] : 置位
>
> [4] : 将变量 cr0 的值赋给 CR0 系统寄存器
>
> 注释 2：
>
> volatile : No reordering, no elimination
>
> %0 : the first constraint(约束) following
>
> r : A constraint, GCC is free to use any register ,r代表任意寄存器

```C
[1] uint32_t cr0;//cr0内存变量
[2] asm volatile ("movl %%cr0, %0\n" : "=r"(cr0));
[3] cr0 |= 0x80000000;
[4] asm volatile ("movl %0, %%cr0\n"::"r"(cr0));
```

Generated assembly code (*.S)

> 生成的汇编代码

```assembly
movl %cr0, %ebx
movl %ebx, 12(%esp)
orl  $-2147483648, 12(%esp)
movl 12(%esp), %eax
movl %eax, %cr0
```

### Example 2

Inline assembly

```C
long res,arg1=2,arg2=22,arg3=222,arg4=223;
asm volatile("int $0x80"
            :"=a"(res)
            :"0"(11),"b"(arg1),"c"(arg2),"d"(arg3),"S"(arg4));
```



















