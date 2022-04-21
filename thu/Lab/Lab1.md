# Lab 1

> bootloader 启动 OS

[TOC]



# 知识准备



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

### 基本内联汇编



### 扩展内联汇编

#### `Grammar` :

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

#### `Tips` :

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

#### `Output operand list` :

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

#### `Input operand list` :

>1. 输入部分与输出部分相似，但没有“＝”。
>2. 如果输入部分一个操作数所要求使用的寄存器，与前面输出部分某个约束所要求的是同一个寄存器，那就把对应操作数的编号（如“1”，“2”等）放在约束条件中。



#### `clobber list` :

> 1. 也称为乱码列表
> 2. 这部分常常以“memory”为约束条件，以表示操作完成后内存中的内容已有改变，如果原来某个寄存器的内容来自内存，那么现在内存中这个单元的内容已经改变。乱码列表通知编译器，有些寄存器或内存因内联汇编块造成乱码，可隐式地破坏了条件寄存器的某些位（字段）。

1. 当输入部分存在，而输出部分不存在时，分号“：“要保留
2. 当“memory”存在时，三个分号都要保留

```C
#define __cli() __asm__ __volatile__("cli": : :"memory")
```



#### example 1

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

#### example 2

Inline assembly

```C
long res,arg1=2,arg2=22,arg3=222,arg4=223;
asm volatile("int $0x80"
            :"=a"(res)
            :"0"(11),"b"(arg1),"c"(arg2),"d"(arg3),"S"(arg4));
```

1. `Output` : res <-- %eax

2. `Input` : 注意输入行 `"0"`也表示eax

   > 顺序：
   >
   > q 指示编译器从eax, ebx, ecx, edx分配寄存器
   >
   > r 指示编译器从eax, ebx, ecx, edx, esi, edi分配寄存器
   >
   > * 数字%n的用法：数字表示的寄存器是按照出现和从左到右的顺序映射到用"r"或"q"请求的寄存器．
   >
   > * 如果要重用"r"或"q"请求的寄存器的话，就可以使用它们。
   > * 此处重用了`%eax`，因此根据上所示顺序, "0"即对应%eax

   1. $11 -> %eax
   2. arg1 -> %ebx
   3. arg2 -> %ecx
   4. arg3 -> %edx
   5. arg4 -> %esi

#### example 3

Inline assembly

```C
   int count=1;
    int value=1;
    int buf[10];
    void main()
    {
        asm(
            "cld nt"
            "rep nt"
            "stosl"
        :
        : "c" (count), "a" (value) , "D" (buf[0])
        : "%ecx","%edi"
        );
    }
```

1. 冒号后的语句依次指明输出`Output`，输入`Input`和被改变的寄存器`clobbers`。
2. cld,rep,stos这几条语句的功能是向buf中写上count个value值。

Assembly

```assembly
    movl count,%ecx
    movl value,%eax
    movl buf,%edi
    #APP
    cld
    rep
    stosl
    #NO_APP
```

#### example 4

> 可以让gcc自己选择合适的寄存器

inline as 

```C
 asm("leal (%1,%1,4),%0"
        : "=r" (x)
        : "0" (x)
    );
```

1. `Onput` : x <-- %eax
2. `Iutput` : x --> %eax
3. 注意顺序

assmebly

```assembly
movl x,%eax
#APP
leal (%eax,%eax,4),%eax
#NO_APP
movl %eax,x
```

- [1] 使用q指示编译器从eax, ebx, ecx, edx分配寄存器。 使用r指示编译器从eax, ebx, ecx, edx, esi, edi分配寄存器。

- [2] 不必把编译器分配的寄存器放入改变的寄存器列表，因为寄存器已经记住了它们。

- [3] "="是标示输出寄存器，必须这样用。

- [4] 数字%n的用法：数字表示的寄存器是按照出现和从左到右的顺序映射到用"r"或"q"请求的寄存器．如果要重用"r"或"q"请求的寄存器的话，就可以使用它们。

- [5] 如果强制使用固定的寄存器的话，如不用%1，而用ebx，则：

  ```
    asm("leal (%%ebx,%%ebx,4),%0"
        : "=r" (x)
        : "0" (x) 
    );
  ```



## x86的中断处理

> 1. x86 中断源
> 2. CPU和OS如何处理中断
> 3. 能够对中断向量表（中断描述符表，IDT）初始化

 

### x86中断处理：确定中断服务例程

* 每个中断/异常与一个中断服务例程`ISR(Interrupt Service Routine)`关联，其关联关系储存在中断描述符表`IDT(Interrupt Descriptor Table)`
* `IDT`的起始地址和大小保存在中断描述符表寄存器`IDTR`中
  * `IDT`需要`OS`建立
  * `IDTR`需要`OS`一个指令告诉`CPU`
* `ucore`建立好两个表：GDT，IDT
  * CPU基于这两个表将某次中断和相应的中断服务例程联系起来
  * 异常/中断 ---> IDT ---> GDT ---> ISR
* 段描述符会设定响应的特权级
  * CS : 低两位描述特权级：00(内核态), 01, 10, 11(用户态)
* 不同特权级的中断切换对堆栈的影响
  * 内核态中断：中断处理仍在内核态
  * 用户态中断：跳转到内核态处理中断（产生特权级变化）

### x86中断处理：从中断服务例程返回

* `iret`完成中断例程返回

  * 弹出`eflags`和`ss/esp`(改变特权级就会改变栈，所以返回时要恢复ss和esp)

  * 特权级变化情况下返回时弹出的内容:

    EIP, CS, EFLAGS, ESP, SS

* `ret` or `retf`函数调用返回 

  * `ret`弹出`EIP`
  * 远程跳转：`retf`弹出`CS`和`EIP`

* 以上都是硬件实现的



### x86的中断处理：系统调用

* 用户程序通过系统调用访问`OS kernel`服务
* 实现
  * 指定中断号
  * 使用 `trap`  (软中断)，Software generated interrupt
  * 或者使用特殊指令`sysenter/sysexit`





# 代码



## 练习一

> 1. 操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)
> 2. 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

### 涉及内容

* 外设：串口，并口，CGA，时钟，硬盘
* Bootloader 软件
* ucore OS 软件

### 示例

> 基于lab1_result

* 执行

  ```makefile
  make clean
  ```

* 参数`"V="`使得编译执行的详细过程会展现出来

  > 注意这个参数是`Makefile`自定义的不是make的内置参数

  ```makefile
  make V=
  ```

  

### 环境变量

> 查看`Makefile`内容

#### makefile function 语法

```makefile
$(<function> <arguments>)
or
${<function> <arguments>}
```

* 这里， `<function>` 就是函数名，make支持的函数不多。
*  `<arguments>` 为函数的参数，参数间以逗号 `,` 分隔，而函数名和参数之间以“空格”分隔。函数调用以 `$` 开头，以圆括号或花括号把函数名和参数括起。
* 函数调用以 `$` 开头，以圆括号或花括号把函数名和参数括起。

#### call 函数

* call函数是唯一一个可以用来创建新的参数化的函数。你可以写一个非常复杂的表达式，这个表达式中，你可以定义许多参数，然后你可以call函数来向这个表达式传递参数。其语法是：

  ```makefile
  $(call <expression>,<parm1>,<parm2>,...,<parmn>)
  ```

  当make执行这个函数时， `<expression>` 参数中的变量，如 `$(1)` 、 `$(2)` 等，会被参数 `<parm1>` 、 `<parm2>` 、 `<parm3>` 依次取代。而 `<expression>` 的返回值就是 call 函数的返回值。例如：

  ```makefile
  reverse =  $(1) $(2)
  
  foo = $(call reverse,a,b)
  ```

  那么， `foo` 的值就是 `a b` 。当然，参数的次序是可以自定义的，不一定是顺序的，如：

  ```makefile
  reverse =  $(2) $(1)
  
  foo = $(call reverse,a,b)
  ```

  此时的 `foo` 的值就是 `b a` 。

* 需要注意：在向 call 函数传递参数时要尤其注意**空格**的使用。call 函数在处理参数时，**第2个及其之后的参数中的空格会被保留，因而可能造成一些奇怪的效果**。因而在向call函数提供参数时，最安全的做法是去除所有多余的空格。





### Makefile详解

* **line-117** : 生成`libs`目录下的`obj`变量名

  ```makefile
  $(call add_files_cc,$(call listf_cc,$(LIBDIR)),libs,)
  ```

  1. 调用函数`add_file_cc`

     输入参数：2个，第一个是调用函数`listf_cc`的返回值，第二个是目录`libs`

  2. `listf_cc`函数定义为 line-89 处的

     ```makefile
      [89] listf_cc = $(call listf,$(1),$(CTYPE))
     ```

     listf的参数有2个，第一个是call listf_cc传入的参数，也就是`$(1) = $(LIBDIR) = `，第二个参数为`$(CTYPE)`

  3. 





* **line-136** :

  ```makefile
  $(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
  ```

  





### Q1 ucore.img 的生成

> 需要`kernel`和`bootblock`

**示意图**

> 基于`Makefile`流程

```mermaid
flowchart LR
    tools -->| sign.c | O1[bin/sign]
   
    boot -->| -r *.c, *.S | O2[*.o]
    	O2 --> | ld | P2[bin/bootblock.out]
    	P2 --> tmp2( )
    	
    O1 --> tmp2
    tmp2 --> Q1(bin/bootblock :512bytes)
    
    kern --> | *.c, *.S | O3[*.o]
    libs --> | *.c, *.S | O3
    O3 --> | ld | P3[bin/kernel]
    
    P3 --> RES{ucore.img}
    Q1 --> RES
```































