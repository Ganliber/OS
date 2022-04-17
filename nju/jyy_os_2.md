# Course : NJU

> 相关代码放在 pwn 库的 docker_env 的OS文件夹下
>
> 课程主页地址 ：[jyywiki](http://jyywiki.cn/OS/2022/)

[TOC]



## P6 并发控制 1 : 互斥

### 共享内存 : 互斥

* 互斥 `mutual exclusion`

  * 实现`lock_t`数据结构和`lock/unlock`API

    ```C
    typedef struct{
      ...
    }lock_t;
    void lock(lock_t * lk); --- 试图获得锁的独占访问,成功获得后返回
    void unlock(lock_t * lk); --- 释放锁的独占访问
    ```

  * 一把 **排他性** 的锁 : 对于锁对象`lk`

    * 在任何线程调度（线程执行的顺序）下
      * 若某`thread`持有`lk` ：`lock(lk)`返回且未释放
      * 那么任何`other threads` 的`lock(lk)`都不能返回
    * p.s.状态机视角
      * lock返回会进入`locked`的状态，`unlock`会清除该状态
      * 初始状态`s0`**不能到达两个线程都进入`locked`的状态**（也就是P5中最后例子中的`safety`[3]）
    * 保证原子性，顺序性，可见性
      * 在**共享内存**上，共享资源的访问太危险（原子性，顺序性，可见性的丧失），互斥`mutual exclusion`用来实现代码块之间的并发，实现"串行化"
        * `lock/unlock`保护区域成为一个**原子**的黑盒子
        * 黑盒子的代码不能随意并发，**顺序**满足要么T1->T2, 要么T2->T1
        * 且先完成的黑盒子的内存访问在之后访问的黑盒子中可见

* 共享内存多线程 : 独立的`registers/stack` , 共享内存`heap, static/global variable, .test`

  > 注：堆是线程共享的内存区域，栈是线程独享的内存区域。堆中主要存放对象实例，栈中主要存放各种基本数据类型、对象的引用。

  * 支持的基本操作

    * `thread-local`线程本地计算 : registers/value read, written and modified on stack
    * `load` : read shared memory
    * `store` : write shared memory

  * 假设：

    * **旧的假设**：基于**指令**原子性（Peterson算法（实际行不通）），但实际上`x86`每一条指令会分成若干个小指令执行,即一条指令由一个微程序实现，而这个微程序中可能包含多次的读/写/本地计算的操作，CPU每个只能执行一条读/写/本地计算，这个是串行的，而对于指令的执行可能是乱序的（实际上在`x86`平台上指令是有可能乱序执行的（为了性能）

    * **新的假设**：基于**读/写/本地计算**的原子性，即假设`load/store`是`atomicity` ：状态机每次只执行一条`load/store`

    * 例子：实际上一条`addq $0x1,xxx(%rip)`可以看成是`t=load(x),t++,store(x,t)`其中包含一条读指令，一条写指令和一条线程本地计算的指令

  * 共享内存上实现互斥的困难

    * load(环顾四周)：不能写，只能 “干看”

    * store(改变物理世界的状态)：不能读，只能 “闭着眼睛动手”

    * (现代多处理器) load/store 可能在执行时被**乱序**执行

    * 软件无法实现的问题，硬件可以来处理

    * 示例1（有问题）

      ```C
      void critical_section()
      {
        while(1)
        {
      [1]    if(!locked){											---  load  ---
      [2]      locked=1;											---  store ---
      [3]      break;
             }
          //critical section
      [4]    locked=0;
        }
      ```

      实际上问题在于当两个线程T1, T2, 设(PC1, PC2, locked), 会出现干扰导致两个程序同时进入临界区，这样一来就无法实现互斥，而现实世界却为能不被打断的同时`load + store`所以需要[1]和[2]同时完成才可以实现

      ```
      (1,1,0)
      + ---T1--- (2,1,0)
      	+ ---T1--- (3,1,1)
      	+ ---T2--- (2,2,0) ;safety is destroyed
      		+ ---T1--- (3,2,1)
      			+ ---T2--- (3,3,1) ;同时上锁,同时进入临界区
      		+ ---T2--- (2,3,1)
      			+ ---T1--- (3,3,1) ;同时上锁,同时进入临界区
      ```

      改进目的：一步完成`[1]+[2]`（一边看着门有没有被锁一边锁上门），这样就可以避免上述现象（干扰）的出现。

      * 为了实现互斥锁，需要**同时完成：**
        1. t = load(locked)
        2. if(t==0) store(locked,1)
      * `test-and-set` （原子指令）
      * 通过硬件解决

### 原子指令

--- 或称为原子操作

> 一条不可分割的指令，该类指令能够完成：
>
> 1. 一次共享内存的`load`
> 2. 向同一个共享内存地址的`store`
> 3. 一些线程(处理器)本地的计算

* 多处理器，原子指令保证：
  * 原子性：`load/store` will not be interrupt
  * 顺序：线程（处理器）执行的乱序不能越过原子操作
  * 多处理器之间的可见性：若原子操作A发生在B前，则A之前的`store`对B之后的`load`可见

* 原子指令会导致运行时间大增

* `asm`内联汇编可以用lock前缀锁定某个指令不被处理器 "肢解"

* `x86`原子操作：

  1. `LOCK`指令前缀

  2. `xchg`（实际上是硬件实现的，代码只是对硬件的一种模拟）

     对`xchg`的封装，功能：实现将addr处储存的值和newval交换（即返回旧值，addr为新值）

     ```C
     int xchg(volatile int * addr, int newval)
     {
       int result;
       asm volatile("lock xchg %0,%1"
                    :"+m"(*addr), //[%0] addr (内存:读写)
                     "=a"(result) //[%1] result (%eax)
                    :"1"(newval)  //[%1]
                    :"cc");       //clobbers(破坏者) eflags
       return result;
     }
     ```

#### 互斥实现：自旋锁

--- `spin lock`

* 实现

  ```C
  int table = KEY; ---table <=> locked
  void lock(){
    while(1){
      int got = xchg(&table,NOTE);
      if(got == KEY) break;
  }}
  void unlock(){
    xchg(&table, KEY);
  }
  ```

* 重构代码

  ```c
  int locked = 0;//桌子就是锁,设纸条就是1
  void lock(){
    while(xchg(&locked,1));
  }
  void unlock(){
    xchg(&lock,0);
  }
  ```

#### RISC-V : 另一种原子操作的设计

考虑原子操作：

* `atomic test-and-set`

  ```
  reg = load(x) ;
  if (reg == XX) { store(x, YY); }
  ```

* `lock xchg`

  ```
  reg = load(x) ;
  store(x, XX);
  ```

* `lock add`

  ```
  t = load(x); 
  t++; 
  store(x, t);
  ```

常用原子指令的本质：

1. `load(x)`

2. 设置处理器局部`registers`状态
3. `store(x)`

补充：实现对于一对`load - store`，在中间可以实现任意的`thread-local`操作，而保证这对`load - store`的原子性，顺序性，可见性

##### LR/SC

> 在riscv这样处理器中提供的机制
>
> Load-Reserved/Store-Conditional

* Load Reserved
* Store Conditional

### Data Race

* 数据竞争：两个不同的线程的两个操作访问**同一段内存**，且至少有一个是`store`，**其中没有原子操作间隔**
* 数据竞争 <=> undefined behavior
* 编程时应彻底避免数据竞争
  * 充分条件：！！！！！！使所有线程间共享的变量都被**同一把互斥锁**保护

### 本节知识补充

* CPU的实现
  * CPU = 数字逻辑电路
    * 寄存器 + 组合逻辑
  * 硬件描述语言( HDL, Hardware Description Language )
    * Verilog HDL, VHDL
    * Chisel (on Scala)



## P7 代码：硬件眼里的操作系统(未完)

> ！！！！！！！！！！暂时还无法实现完全理解！！！！！！！！！！
>
> 1. AbstractMachine
> 2. 软硬件桥梁
> 3. 操作系统：是个C程序
> 4. 操作系统：是个状态机

* 通往专业之路

  > OS 已经是一个很成熟的领域

  * 顶级研究论文：OSDI, SOSP, ATC, EuroSys, ...
  * 久经考验的教学材料：xv6, OSTEP, CSAPP, ...
  * 海量的开源工具：GNU系列, qemu, gdb, ...
  * 第三方资料慎用：tutorials, osdev wiki, ...

* "写一个操作系统"对于新手最困难的部分，恰恰与操作系统没什么关系

  * 硬件提供的复杂机制
    * GDT, IDT, TSS, ... 但这些概念并非需要掌握
  * 虚拟机上的程序很难调试
    * 汇编
    * 编程基础
    * 工具使用
    * 缺少好用的工具

### C语言运行环境

* C语言编译/链接成ELF目标文件命令
  * makefile, ld script, ...
* 二进制文件在`bare-metal`（裸机）上运行必要的其他部件
  * AM`Abstract Machine` API
  * ELF 文件的加载器（代码，编译/链接脚本）
* 生成系统镜像文件的脚本
* 平台相关的启动脚本
  * 启动虚拟机执行
  * 将镜像写入开发板并启动

#### 主要使用工具

* gcc GNU binutils : a collection of binary tools
* ld（链接器）, as（汇编器）, ...
* ar（静态库打包）, objcopy（目标文件解析）, ...
* 其他工具：nm, strings, size, objdump, readelf, ...

### Bare-Metal 与程序员的约定

> 裸机（硬件）和程序员（软件编写者）的约定

* 为了能运行我们的程序，硬软件的约定

  > CPU reset 之后，从PC pointer 取指令，译码，执行...
  >
  > 从`firmware`开始执行，注意，`0xffff0`通常是一条向`firmware`跳转的`jmp`指令

  * `CPU reset`（CPU复位）之后，处理器处于某个确定的状态

    * PC pointer 指向一段 memory-mapped ROM
      * ROM 储存量厂商提供的 firmware (固件)
    * 处理器大部分特性处于关闭状态
      * 缓存，虚拟储存，...

  * `Firmware`

    * 关于`firmware`:`BIOS` v.s. `UEFI`
      * 都是主板/主板上外插设备的软件抽象
        * 支持系统管理程序的运行
      * BIOS 属于老式的版本，UEFI较为复杂
      * Legacy BIOS ( Basic I/O System ) : legacy(译：旧版的，遗产)
      * UEFI ( Unified Extensible Firmware Interface ) ：统一可扩展固件接口（英语：Unified Extensible Firmware Interface，缩写UEFI）是一种个人电脑系统规格，用来定义操作系统与系统固件之间的软件界面，作为BIOS的替代方案。 可扩展固件接口负责加电自检（POST）、联系操作系统以及提供连接操作系统与硬件的接口。
    * 将用户数据加载到内存
      * 储存介质上的第二级`loader`
      * 或者直接加载操作系统`嵌入式系统`
    * `U-Boot` : the universal boot loader
      * 解释：固件`firmware`加载`U-Boot`, `U-boot`加载各种各样的操作系统

  * Legacy BIOS : 约定

    > Firmware 必须提供机制，将用户数据载入内存

    * Legacy BIOS 把引导盘的第一个扇区（主引导扇区，MBR）加载到内存的`7c00`位置
      * 处理器为`16-bit`
      * 规定 CS:IP = 0x7c00

### QEMU

> 世界上应用最广泛的开源虚拟机

* 传奇黑客，天才程序员 Fabrice Bellard 的杰作
  * Android Virtual Device, Virtualbox, ...
* `QEMU`, A fast and portable dynamic translator `USENIX ATC'05`
  * `SeaBIOS` : QEMU `x86` 的默认 `firmware`
    * SeaBIOS 会被编译成一个 `ELF`二进制文件 

### 用脚本处理读取数据

> 意识

### 本节知识补充



## P8 并发控制 2 : 操作系统中的互斥

> **xv6 spinlock** 

















