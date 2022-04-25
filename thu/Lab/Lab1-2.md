# Lab 1-2

> 由于一个文件写到最后太卡了（`Typora`rnm退钱)，就再开一个

[TOC]

## 练习 3

> BIOS将通过读取硬盘主引导扇区到内存，并转跳到对应内存中的位置`0x7c00`执行bootloader。请分析bootloader是如何完成从实模式进入保护模式的。
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
     注释:
     25 # 启用 A20:
     26 # 为了向后兼容最早的 PC，物理
     27 # 地址线A20系低，以至于地址高于
     28 # 1MB 默认归零。 此代码撤消了此操作。
```

步骤

1. 等待 8042 Input buffer为空 [30]
2. 发送 Write 8042 Output Port （P2） 命令到8042 Input buffer
3. 等待 8042 Input buffer为空
4. 将 8042 Output Port（P2） 对应字节的第2位置1，然后写入8042 Input buffer

然后是第一段`seta20.1`

```assembly
循环体 : 等待 8042 input buffer 为 0
     29 seta20.1:
     30     inb $0x64, %al                           # Wait for not busy(8042 input buffer empty).
     +: 往端口0x64写数据
     31     testb $0x2, %al
     +: (逻辑与&)测试%al的值
     32     jnz seta20.1
     +: 如果 %al 的从右数第2位为1 跳转回 seta20.1
     

     34     movb $0xd1, %al                          # 0xd1 -> port 0x64
     +: 先将 0xd1 存入 %al 中
     35     outb %al, $0x64                          # 0xd1 means: write data to 8042's P2 port
     +: 再将该数据写入 0x64 port
     
 结果 : seta20.1是往端口0x64写数据0xd1，告诉CPU我要往8042芯片的P2端口写数据
```

1. `in`和`out`指令

   * 汇编语言中，CPU对外设的操作通过专门的端口读写指令来完成；读端口用IN指令，写端口用OUT指令。
   * 汇编是直接面向硬件的，它可以访问系统的mem空间，也可以直接访问系统的io空间。汇编中使用in/out来访问系统的io空间。

   ```assembly
   　　IN AL,21H	
   　　	+: 表示从21H端口读取一字节数据到AL
   　　IN AX,21H	
   　　	+: 表示从端口地址21H读取1字节数据到AL，从端口地址22H读取1字节到AH
   　　MOV DX,379H
   　　IN AL,DX 
   　　	+: 从端口379H读取1字节到AL
   　　OUT 21H,AL
   　　	+: 将AL的值写入21H端口
   　　OUT 21H,AX
   　　	+: 将AX的值写入端口地址21H开始的连续两个字节。（port[21H]=AL,port[22h]=AH）
   　　MOV DX,378H
   　　OUT DX,AX 
   　　	+: 将AH和AL分别写入端口379H和378H
   ```

2. `test`指令

   测试一个位

   ```assembly
   test eax, 100b; b后缀意为二进制
   jnz **; 如果eax右数第三个位为1,jnz将会跳转
   ```

   测试一方寄存器是否为空:

   ```assembly
   test ecx, ecx
   jz somewhere
   ```

   如果ecx为零,设置ZF零标志为1,Jz跳转



第二段`seta20.2`

```assembly
循环体 : 等待seta20.2的input buffer为0,目的是确保输入缓冲区为空,这样才能保证后续写操作的有效性
     37 seta20.2:
     38     inb $0x64, %al             # Wait for not busy(8042 input buffer empty).
     39     testb $0x2, %al
     40     jnz seta20.2


     42     movb $0xdf, %al            # 0xdf -> port 0x60
     +: 将 0xdf 暂时存入 %al中
     43     outb %al, $0x60            # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
     +: 写入 0x60 port, 0xdf== 11011111
     
结果 : seta20.2是往端口0x60写数据0xdf，从而将8042芯片的P2端口设置为1. 
```



### Q2

#### 初始化GDT表

> `gdtdesc : ...`描述了GDT的 `size` 和 `address`
>
> `gdt : ...`描述了GDT的具体内容：null，code，data

依然在`boot/bootasm.S`先看注释

```
     45     # Switch from real to protected mode, using a bootstrap GDT
     46     # and segment translation that makes virtual addresses
     47     # identical to physical addresses, so that the
     48     # effective memory map does not change during the switch.
```



##### GDT加载

```assembly
     49     lgdt gdtdesc
```

* `LGDT/LIDT `- 加载全局/中断描述符表格寄存器，LGDT 与LIDT 指令仅用在操作系统软件中；它们不用在应用程序中。在保护模式中，它们是**仅有的能够直接加载线性地址**（即，不是段相对地址）与限制的指令。 它们通常在实地址模式中执行，以便处理器在切换到保护模式之前进行初始化。

  | 操作码   | 指令            | 说明                 |
  | -------- | --------------- | -------------------- |
  | 0F 01 /2 | LGDT **m16&32** | 将 **m** 加载到 GDTR |
  | 0F 01 /3 | LIDT **m16&32** | 将 **m** 加载到 IDTR |

* `lgdt gdtdesc`把全局描述符表的大小和起始地址共**8个字节（64-bit）**加载到全局描述符表寄存器GDTR中

  ```assembly
       84 gdtdesc:
       85     .word 0x17                                      # sizeof(gdt) - 1
       86     .long gdt                                       # address gdt
  ```

  `gdtdesc`描述了GDT的 `size` 和 `address`

  大小为 sizeof(gdt) = 0x17 + 1 = 0x18 = (24)~10~ bytes = 3 * 8bytes，三项，每一项 8 bytes ( 64-bit )



##### GDT内容

```assembly
     77 # Bootstrap GDT
     78 .p2align 2                                          # force 4 byte alignment
     79 gdt:
     80     SEG_NULLASM                                     # null seg
     +: null 空白项
     81     SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
     +: code segment 可读可执行
     +: 0 ~ 2**32 (4G)
     82     SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel
     +: data segment 可写
```

* 共有3项，每项8字节。
  * 第1项是空白项，内容为全0. 
  * 后面2项分别是**代码段**和**数据段**的描述符，它们的base都设置为`0x0`，limit都设置为`0xffffff`，也就是长度均为4G.
* 由于全局描述符表每项大小为8字节，因此一共有3项，而第一项是空白项，所以全局描述符表中只有两个有效的段描述符，分别对应**代码段**和**数据段**。

##### 查看 SEG_ASM 宏

在`boot/asm.h`中

```C
     11 #define SEG_ASM(type,base,lim)                                  \
     12     .word (((lim) >> 12) & 0xffff), ((base) & 0xffff);          \
     13     .byte (((base) >> 16) & 0xff), (0x90 | (type)),             \
     14         (0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)
```





##### GDT的结构

> **全局描述符表** **(GDT)** 是一个从 [Intel](https://zh.wikipedia.org/wiki/英特尔) [x86](https://zh.wikipedia.org/wiki/X86)-系列处理器 [80286](https://zh.wikipedia.org/wiki/Intel_80286) 开始用于界定不同内存区域的特征的数据结构。 全局描述表位于内存中。全局描述表的条目描述及规定了不同内存分区的各种特征，包括基地址、大小和访问等特权如可执行和可写等。 在 Intel 的术语中，这些内存区域被称为 *[段](https://zh.wikipedia.org/wiki/X86記憶體區段)* 。

! 下图展示了`GDT`每一项的结构

图一 （从右下方往左上方看）

![GDT](D:\github\OS\thu\images\GDT.png)

图二（从右上方往左下方看）

<img src="D:\github\OS\thu\images\GDT_Entry.png" alt="GDT_Entry" style="zoom:80%;" />

* GDT每一项有8字节，也就是 64-bit

* 对于`Access Byte`和`Flags`的结构

  ![Gdt_bits](D:\github\OS\thu\images\Gdt_bits.png)

  * `Access Byte` 中 8 个位的含义：

    - *Ac* 是访问位，只要设置为 0 即可。 CPU 会在第一次访问这个段之后设置为 1 。

    - RW 是读写位：
      - 对于数据段： 0 表示不可写（只读）， 1 表示可写。数据段永远可读。
      - 对于代码段： 0 表示不可读（无访问权限）， 1 表示可读。代码段永远不可写。

    - DC位， Direction/Conforming 位：
      - 对于数据段： 0 表示访问该段的偏移必须要比 Limit 要大（向下增长）， 1 则是要小（向上增长）。
      - 对于代码段： 1 表示可以被更低运行权限（更大 Privl ）执行， 0 表示只能被同权或更高权限执行。

    - *Ex* 是可执行位： 0 表示该段为数据段， 1 表示该段为代码段。

    - *Privl* 是权限位，占据两个比特。权限位从 0~3 ，越小表示权限越高，最高为 0 。

    - *Pr* 为 Present 位，对于所有可用段必须为 1 。

  * `Flags` 中的 4 个比特含义：

    - *Sz* 是大小位： 0 表示该段运行于 16-bit 保护模式下， 1 表示运行于 32-bit 保护模式下。用于向后兼容。

    - *Gr* 是粒度位： 0 表示 Limit 的单位是 1 Byte ， 1 表示 Limit 的单位是 4 KB （一个页）。

    - *Flags* 中的第 2 个 bit 在 x86-64 中指明为 64 位描述符，此时 Sz 位必须为 0 。

  * 现在` ucore `**只有代码段和数据段**，所以只设置了这两个。





### Q3

#### Enable protected mode

> **如何使能和进入保护模式**
>
> x86 引入了几个新的**控制寄存器 (Control Registers)** `cr0` `cr1` … `cr7` ，每个长 32 位。这其中的某些寄存器的某些位被用来控制 CPU 的工作模式，其中 `cr0` 的最低位，就是用来控制 CPU 是否处于保护模式的。

查看代码

```assembly
     50     movl %cr0, %eax
     51     orl $CR0_PE_ON, %eax
     52     movl %eax, %cr0
```

* **将cr0寄存器的PE位（cr0寄存器的最低位）设置为1，便使能和进入保护模式了**
* 因为控制寄存器不能直接拿来运算，所以需要通过通用寄存器`%eax`来进行一次存取，设置 `cr0` 最低位为 1 之后就已经进入保护模式。
* 但是最后，由于一些现代 CPU 特性 （乱序执行和分支预测等），在转到保护模式之后 CPU 可能仍然在跑着实模式下的代码，这显然会造成一些问题。因此必须强迫 CPU 清空一次缓冲。对此，最有效的方法就是进行一次 long jump 。

其中 `$CR0_PE_ON`的值的定义

```assembly
     10 .set CR0_PE_ON,             0x1                     # protected mode enable flag
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

### Q1

> bootloader如何读取硬盘扇区

### 硬件访问

> 1. 考虑到实现的简单性，bootloader的访问硬盘都是LBA模式的PIO（Program IO）方式，即**所有的IO操作是通过CPU访问硬盘的IO地址寄存器完成。**
> 2. 当前硬盘数据是储存到硬盘扇区中，一个扇区大小为512字节。读一个扇区的流程（可参看boot/bootmain.c中的readsect函数实现）大致如下：
>    1. 等待磁盘准备好
>    2. 发出读取扇区的命令
>    3. 等待磁盘准备好
>    4. 把磁盘扇区数据读到指定内存

* 查看`boot/bootmain.c`

  调用关系

  ```
  bootmain()
  :			the entry of bootloader
  |
  + --- readseg(uintptr_t va, uint32_t count, uint32_t offset)                 
  :			read @count bytes at @offset from kernel into virtual address @va, might copy more than 
  :			asked.
  |
  + --- readsect(void *dst, uint32_t secno)
  :			read a single sector at @secno into @dst
  ```

  

  先补充一个表：**磁盘IO地址和对应功能**

  >  第6位：为1=LBA模式；0 = CHS模式 第7位和第5位必须为1

  | IO地址 | 功能                                                         |
  | ------ | ------------------------------------------------------------ |
  | 0x1f0  | 读数据，当0x1f7不为忙状态时，可以读。                        |
  | 0x1f2  | 要读写的扇区数，每次读写前，你需要表明你要读写几个扇区。最小是1个扇区 |
  | 0x1f3  | 如果是LBA模式，就是LBA参数的0-7位                            |
  | 0x1f4  | 如果是LBA模式，就是LBA参数的8-15位                           |
  | 0x1f5  | 如果是LBA模式，就是LBA参数的16-23位                          |
  | 0x1f6  | 第0~3位：如果是LBA模式就是24-27位 第4位：为0主盘；为1从盘    |
  | 0x1f7  | 状态和命令寄存器。操作时先给命令，再读取，如果不是忙状态就从0x1f0端口读数据 |

  `func : readsect`代码

  ```C
   43 /* readsect - read a single sector at @secno into @dst */
   44 static void
   45 readsect(void *dst, uint32_t secno) {
   46     // wait for disk to be ready
   47     waitdisk();
   48 
   49     outb(0x1F2, 1);                         // count = 1
   50     outb(0x1F3, secno & 0xFF);
   51     outb(0x1F4, (secno >> 8) & 0xFF);
   52     outb(0x1F5, (secno >> 16) & 0xFF);
   53     outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
   54     outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors
   55 
   56     // wait for disk to be ready
   57     waitdisk();
   58 
   59     // read a sector
   60     insl(0x1F0, dst, SECTSIZE / 4);
   61 }
  ```

  `func : waitdisk`代码 : 

  > 功能：等待磁盘准备好
  >
  > 0xC0 = 1100 0000b
  >
  > 0x40 = 0100 0000b
  >
  > 不断查询读`0x1F7`寄存器的最高两位，直到最高位为0、次高位为1（这个状态应该意味着磁盘空闲）才返回。

  ```C
       36 /* waitdisk - wait for disk ready */
       37 static void
       38 waitdisk(void) {
       39     while ((inb(0x1F7) & 0xC0) != 0x40)
       40         /* do nothing */;
       41 }
  ```

  发出读取扇区的命令

  ```C
  outb(0x1F2, 1);                         // count = 1
  +: 读写的扇区数为1,放在0x1F2寄存器中
  
  outb(0x1F3, secno & 0xFF);
  outb(0x1F4, (secno >> 8) & 0xFF);
  outb(0x1F5, (secno >> 16) & 0xFF);
  outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
  +: 读取的扇区起始编号共28位，分成4部分依次放在0x1F3~0x1F6寄存器中。
  
  outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors
  +: 对应的命令字为0x20，放在0x1F7寄存器中
  ```

  再次等待硬盘空闲

  ```C
  waitdisk();
  ```

  把磁盘扇区数据读到指定内存

  ```C
       33 #define SECTSIZE        512
       34 #define ELFHDR          ((struct elfhdr *)0x10000)      // scratch space
  
       60 insl(0x1F0, dst, SECTSIZE / 4);
  ```

  * 开始从`0x1F0`寄存器中读数据。
  * `insl`函数：That function will read cnt dwords from the input port specified by port into the supplied output array addr.

  * `insl`是以dword即4字节为单位的，因此这里SECTIZE需要除以4.
  * `dst`是函数`readsect(void *dst, uint32_t secno)`的参数

### Q2

> bootloader如何加载OS(ELF 格式)
>
> 1. 根据`elfhdr`和`proghdr`的结构描述，bootloader就可以完成对ELF格式的ucore操作系统的加载过程（参见boot/bootmain.c中的bootmain函数）。

查看`bin/kernel`

```shell
# 当前文件位置: lab1
file bin/kernel
bin/kernel: ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), statically linked, not stripped
```

* LSB：Linux标准规范（Linux Standard Base）

#### ELF

* ELF可执行文件类型的`elfhdr`和`proghdr`
* 定义在<elf.h>文件中

```C
struct elfhdr {
  uint magic;  --- must equal ELF_MAGIC
  uchar elf[12];
  ushort type;
  ushort machine;
  uint version;
  uint entry;  --- 程序入口的虚拟地址
  uint phoff;  --- program header 表的位置偏移
  uint shoff;
  uint flags;
  ushort ehsize;
  ushort phentsize;
  ushort phnum; --- program header表中的入口数目
  ushort shentsize;
  ushort shnum;
  ushort shstrndx;
};
```

* program header描述与程序执行直接相关的目标文件结构信息，用来在文件中定位各个段的映像，同时包含其他一些用来为程序创建进程映像所必需的信息。
* 可执行文件的程序头部是一个program header结构的数组， 每个结构描述了一个段或者系统准备程序执行所必需的其它信息。目标文件的 “段” 包含一个或者多个 “节区”（section） ，也就是“段内容（Segment Contents）” 。
* 程序头部仅对于可执行文件和共享目标文件有意义。
* 可执行目标文件在ELF头部的e_phentsize和e_phnum成员中给出其自身程序头部的大小。程序头部的数据结构如下表所示：

```C
struct proghdr {
  uint type;   --- 段类型
  uint offset;  --- 段相对文件头的偏移值 : 定位文件中该段的位置
  uint va;     --- 段的第一个字节将被放到内存中的虚拟地址 : 定位段加载到内存中的位置
  uint pa;
  uint filesz;
  uint memsz;  --- 段在内存映像中占用的字节数 : 加载内容的大小
  uint flags;
  uint align;
};
```

#### `lab1`中的ELF

> 位置：`../lab1/libs/elf.h`

```C
      1 #ifndef __LIBS_ELF_H__
      2 #define __LIBS_ELF_H__
      3 
      4 #include <defs.h>
      5 
      6 #define ELF_MAGIC    0x464C457FU            // "\x7FELF" in little endian
      7 
      8 /* file header */
      9 struct elfhdr {
     10     uint32_t e_magic;     // must equal ELF_MAGIC
     11     uint8_t e_elf[12];
     12     uint16_t e_type;      // 1=relocatable, 2=executable, 3=shared object, 4=core image
     13     uint16_t e_machine;   // 3=x86, 4=68K, etc.
     14     uint32_t e_version;   // file version, always 1
     15     uint32_t e_entry;     // entry point if executable
     16     uint32_t e_phoff;     // file position of program header or 0
     17     uint32_t e_shoff;     // file position of section header or 0
     18     uint32_t e_flags;     // architecture-specific flags, usually 0
     19     uint16_t e_ehsize;    // size of this elf header
     20     uint16_t e_phentsize; // size of an entry in program header
     21     uint16_t e_phnum;     // number of entries in program header or 0
     22     uint16_t e_shentsize; // size of an entry in section header
     23     uint16_t e_shnum;     // number of entries in section header or 0
     24     uint16_t e_shstrndx;  // section number that contains section name strings
     25 };
     26 
     27 /* program section header */
     28 struct proghdr {
     29     uint32_t p_type;   // loadable code or data, dynamic linking info,etc.
     30     uint32_t p_offset; // file offset of segment
     31     uint32_t p_va;     // virtual address to map segment
     32     uint32_t p_pa;     // physical address, not used
     33     uint32_t p_filesz; // size of segment in file
     34     uint32_t p_memsz;  // size of segment in memory (bigger if contains bss）
     35     uint32_t p_flags;  // read/write/execute bits
     36     uint32_t p_align;  // required alignment, invariably hardware page size
     37 };
     38 
     39 #endif /* !__LIBS_ELF_H__ */
```



#### `bootmain()`函数

##### 分析

1. 从硬盘中将`bin/kernel`文件的第一页内容加载到内存地址为`0x10000`的位置，来读取kernel文件的ELF Header信息。

   ```C
        88     // read the 1st page off disk
        89     readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
   ```

   虚拟地址是 0x10000 ，一页是 4 KB 共 8*512 bytes，偏移地址是 0

2. 校验ELF Header的e_magic字段，以确保这是一个ELF文件 `line-91 ~ line-94`

3. 读取ELF Header的e_phoff字段，得到Program Header表的起始地址；读取ELF Header的e_phnum字段，得到Program Header表的元素数目。

   * 关于`uintptr_t`和`intptr_t`

     > 1. 这两个数据类型是ISO C99定义的，具体代码在linux平台的/usr/include/stdint.h头文件中。
     > 2. 在C语言中，任何类型的指针都可以转换为void *类型，并且在将它转换回原来的类型时不会丢失信息。
     > 3. 在64位的机器上，intptr_t和uintptr_t分别是`long int`、`unsigned long int`的别名。
     > 4. 在32位的机器上，intptr_t和uintptr_t分别是`int`、`unsigned int`的别名。
     > 5. 提高程序的可移植性

     ```C
     /* Types for `void *' pointers.  */
     #if __WORDSIZE == 64
     # ifndef __intptr_t_defined
     typedef long int		intptr_t;
     #  define __intptr_t_defined
     # endif
     typedef unsigned long int	uintptr_t;
     #else
     # ifndef __intptr_t_defined
     typedef int			intptr_t;
     #  define __intptr_t_defined
     # endif
     typedef unsigned int		uintptr_t;
     #endif
     ```

   * 代码

     ```C
     ph : proghdr的地址
     eph : proghdr的尾部
     ```

4. 遍历Program Header表中的每个元素，得到每个Segment在文件中的偏移、要加载到内存中的位置（虚拟地址）及Segment的长度等信息，并通过磁盘I/O进行加载

   * 代码`line-101 ~ line-103`

   * 提示

     ```C
     readseg(uintptr_t va, uint32_t count, uint32_t offset)                 
     *** 注释: read @count bytes at @offset from kernel into virtual address @va, might copy more than asked.
     ```

5. 加载完毕，通过ELF Header的e_entry得到`kernel`的入口地址，并跳转到该地址开始执行内核代码

   ```C
       105     // call the entry point from the ELF header
       106     // note: does not return
       107     ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
   ```

##### 代码全貌

```C
     33 #define SECTSIZE        512
     34 #define ELFHDR          ((struct elfhdr *)0x10000)      // scratch space
       
     63 /* *
     64  * readseg - read @count bytes at @offset from kernel into virtual address @va,
     65  * might copy more than asked.
     66  * */
     67 static void
     68 readseg(uintptr_t va, uint32_t count, uint32_t offset){...}

     85 /* bootmain - the entry of bootloader */
     86 void
     87 bootmain(void) {
     88     // read the 1st page off disk
     89     readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
     90 
     91     // is this a valid ELF?
     92     if (ELFHDR->e_magic != ELF_MAGIC) {
     93         goto bad;//见最下方异常处理部分
     94     }
     95 
     96     struct proghdr *ph, *eph;
     97 
     98     // load each program segment (ignores ph flags)
     99     ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
       	    +:e_phoff 是 program header 表的位置偏移
              
    100     eph = ph + ELFHDR->e_phnum;
            +:e_phnum 是 program header表中的入口数目
              
    101     for (; ph < eph; ph ++) {
    102         readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    103     }
            +:遍历每一个 program header 的元素,从kernel文件p_offset偏移处读取p_memsz bytes到内存p_va
    104 
    105     // call the entry point from the ELF header
    106     // note: does not return
    107     ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
    108 
    109 bad:
    110     outw(0x8A00, 0x8A00);
    111     outw(0x8A00, 0x8E00);
    112 
    113     /* do nothing */
    114     while (1);
    115 }
```



#### 调试验证准备

* `make debug`进入

* 在文件`bootasm.S`中，71行处有指令

  ```assembly
      71     call bootmain
  ```

  也是该文件中**唯一**的一个call指令（便于查找）

* 进入调试后首先`b *0x7c00`进入`bootloader`

* 查看从`0x7c00`开始的若干条指令，找到第一个`call`指令

  ![bootmain_entry](D:\github\OS\thu\images\bootmain_entry.png)

  发现`bootmain`的入口地址为`0x7cd1`（该值与机器相关）

* 在`bootmain`处打断点

  ```shell
  break *0x7cd1
  ```

* 进入到`bootmain()`中之后根据`bootmain()`的函数调用次序

  ```
  + --- bootmain
  + --- readseg : read the first page of disk
  + --- for ph:eph {readseg}
  + --- call kernel entry
  ```

  看到第一条`call`指令

  ```assembly
     0x7ce8:	call   0x7c72
     0x7ced:	cmp    $0x9,%ebx
  ```

  在下一条指令处`0x7ced`打断点，`continue`之后表示`OS kernel ELF`的第一页内容加载到内存地址为`0x10000`的位置

  验证`magic`子串

  ```assembly
       10     uint32_t e_magic;     // must equal ELF_MAGIC
  ```

  结果为

  ```
  (gdb) x /xw 0x10000
  0x10000:	0x464c457f
  ```

  `0x464c45`实际上对应字符串"elf"，最低位的`0x7f`字符对应DEL

#### 验证ELF各个字段

* 验证`ELF`中各个字段

  * `ELF Header`

    由于`bootmain()`中有`ELFHDR + ELFHDR->e_phoff`所以可以查看该地址值（在0x10000附近）

    ```assembly
       0x7cfe:	mov    0x1001c,%eax
    ```

    ELF Header的e_phoff字段将加载到eax寄存器，0x1001c相对0x10000的偏移为0x1c，即相差28个字节，这与ELF Header的定义相吻合。

    ```C
                  9 struct elfhdr {
     0 +0        10     uint32_t e_magic;     // must equal ELF_MAGIC
     0 +4        11     uint8_t e_elf[12];
     4 +1*12     12     uint16_t e_type;      
    16 +2        13     uint16_t e_machine;   // 3=x86, 4=68K, etc.
    18 +2        14     uint32_t e_version;   // file version, always 1
    20 +4        15     uint32_t e_entry;     // entry point if executable
    24 +4        16     uint32_t e_phoff;     // file position of program header or 0
    ```

    偏移量正好是`4+12+2+2+4+4 = 28 = 0x1c`

    注意！此处不是立即数赋值，而是将该地址的内容赋值给`%eax`，所以执行完`0x7cfe`后查看`%eax`的值即为`uint32_t e_phoff`即 program header 表的位置偏移

    ```assembly
    (gdb) info register
    eax            0x34	52
    ```

    program header 地址为`0x34`，因此`proghdr`在内存中的位置为 `0x10000 + 0x34 = 0x10034`

  * 查询`0x10034`往后8个字节的内容如下所示：

    ```assembly
    (gdb) x /8xw 0x10034
    0x10034:	0x00000001	0x00001000	0x00100000	0x00100000
    0x10044:	0x0000cfe0	0x0000cfe0	0x00000005	0x00001000
    ```

    与`proghdr`的各个量对照

    ```C
    struct proghdr {
      uint type;   --- 段类型
      uint offset;  --- 段相对文件头的偏移值 : 定位文件中该段的位置
      uint va;     --- 段的第一个字节将被放到内存中的虚拟地址 : 定位段加载到内存中的位置
      uint pa;
      uint filesz;
      uint memsz;  --- 段在内存映像中占用的字节数 : 加载内容的大小
      uint flags;
      uint align;
    };
    ```

    使用`readelf -l bin/kernel`来查询kernel文件各个Segment的基本信息，以作对比。

    ```
    Elf file type is EXEC (Executable file)
    Entry point 0x100000
    There are 3 program headers, starting at offset 52
    
    Program Headers:
      Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
      LOAD           0x001000 0x00100000 0x00100000 0x0cfe0 0x0cfe0 R E 0x1000
      LOAD           0x00e000 0x0010d000 0x0010d000 0x00a16 0x01d20 RW  0x1000
      GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RWE 0x10
    
     Section to Segment mapping:
      Segment Sections...
       00     .text .rodata .stab .stabstr 
       01     .data .bss 
    ```

    注意 type = 1即对应 Program Headers 的第一load行，其中的Flag内容上面为0x5 = 101b,即对应

    R(1)W(0)E(1)：可读可执行

  * 紧跟着`0x1002c`即偏移位置为`0x2c = 44`处，

    ```assembly
       0x7d09:	movzwl 0x1002c,%eax
    ```

    源代码是，此处涉及参数为`e_phnum`

    ```C
      100     eph = ph + ELFHDR->e_phnum;
    ```

    对应`ELF header`，符合

    ```C
    +28     16     uint32_t e_phoff;     // file position of program header or 0
    +32     17     uint32_t e_shoff;     // file position of section header or 0
    +36     18     uint32_t e_flags;     // architecture-specific flags, usually 0
    +40     19     uint16_t e_ehsize;    // size of this elf header
    +42     20     uint16_t e_phentsize; // size of an entry in program header
    +44     21     uint16_t e_phnum;     // number of entries in program header or 0
    ```

    执行之后，eax为3，这说明一共有3个segment。

    ```assembly
    => 0x7d09:	movzwl 0x1002c,%eax
       0x7d10:	shl    $0x5,%eax
    (gdb) si
    0x00007d10 in ?? ()
    (gdb) info r
    eax            0x3	3
    ...
    ```

  * 后面是通过磁盘I/O完成三个Segment的加载，不再赘述。





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

### Stack

```
 + stack bottom direction
                   >>>>> High address
                   
 |  ...            | 
 |  param 3        |
 |  param 2        |
 |  param 1        | 
 |  ret address    | <--- old old EIP
                                         ***** grandparent function
---------------------
 |  old old ebp    | <--- [old ebp]
 |  local variable | 
 |  ...            | 
 |  ...            |
 |  param 3        | =========================================================== ebp[4]
 |  param 2        | =========================================================== ebp[3]
 |  param 1        | =========================================================== ebp[2]
 |  ret address    | <--- old EIP [ebp+4] ====================================== ebp[1]
                                         ***** parent function
---------------------                                             if assume ebp as a pointer(array): 
 |  old ebp        | <--- [ebp] ================================================ ebp[0]
 |  local variable | <--- [ebp-4] (in fact, this means ss:[ebp-4])
 |  ...            |              
                                         ***** child function
                   
                   >>>>> Low address 
 - stack top direction
```

* 从该地址为基准，向上（栈底方向）能获取返回地址、参数值，向下（栈顶方向）能获取函数局部变量值，而该地址处又存储着上一层函数调用时的ebp值。

### Code

```C
void
print_stackframe(void) {
  uint32_t * ebp = 0; // pointer as well as array
  uint32_t eip = 0;
  
  ebp = (uint32_t *) read_ebp();
  eip = read_eip();
 
  while(ebp)
  {
    cprintf("ebp:0x%08x eip:0x%08x args:", (uint32_t)ebp, eip);
    cprintf("0x%08x 0x%08x 0x%08x 0x%08x\n", ebp[2],ebp[3],ebp[4],ebp[5]);
    
    print_debuginfo(eip-1);
    
    eip = ebp[1]; // old EIP
    
    ebp = (uint32_t*)ebp[0];
  }
}
```

* 一些语法

  * `printf`

    ```C
        int PrintVal = 9;
        /*按整型输出，默认右对齐*/
        printf("%d\n",PrintVal);
        /*按整型输出，补齐4位的宽度，补齐位为空格，默认右对齐*/
        printf("%4d\n",PrintVal);
        /*按整形输出，补齐4位的宽度，补齐位为0，默认右对齐*/
        printf("%04d\n",PrintVal);
    
        /*按16进制输出，默认右对齐*/   
        printf("%x\n",PrintVal);
        /*按16进制输出，补齐4位的宽度，补齐位为空格，默认右对齐*/
        printf("%4x\n",PrintVal);
        /*按照16进制输出，补齐4位的宽度，补齐位为0，默认右对齐*/
        printf("%04x\n",PrintVal);
    
        /*按8进制输出，默认右对齐*/
        printf("%o\n",PrintVal);
        /*按8进制输出，补齐4位的宽度，补齐位为空格，默认右对齐*/
        printf("%4o\n",PrintVal);
        /*按照8进制输出，补齐4位的宽度，补齐位为0，默认右对齐*/
        printf("%04o\n",PrintVal);
        
        --- 补齐8位宽度,补齐位置为0,加上0x,16进制输出 ---
        int c =16;	printf("c is 0x%08x", c);
    ```

注释（提出来了）

```
     /* LAB1 YOUR CODE : STEP 1 */
     /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
      * (2) call read_eip() to get the value of eip. the type is (uint32_t);
      * (3) from 0 .. STACKFRAME_DEPTH
      *    (3.1) printf value of ebp, eip
      *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (unit32_t)ebp +2 [0..4]
      *    (3.3) cprintf("\n");
      *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
      *    (3.5) popup a calling stackframe
      *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
      *                   the calling funciton's ebp = ss:[ebp]
      */
```

结果

```
(THU.CST) os is loading ...

Special kernel symbols:
  entry  0x00100000 (phys)
  etext  0x001032d5 (phys)
  edata  0x0010ea16 (phys)
  end    0x0010fd20 (phys)
Kernel executable memory footprint: 64KB
ebp:0x00007b08 eip:0x001009b5 args:0x00010094 0x00000000 0x00007b38 0x00100092
    kern/debug/kdebug.c:309: print_stackframe+36
ebp:0x00007b18 eip:0x00100c81 args:0x00000000 0x00000000 0x00000000 0x00007b88
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b38 eip:0x00100092 args:0x00000000 0x00007b60 0xffff0000 0x00007b64
    kern/init/init.c:48: grade_backtrace2+33
ebp:0x00007b58 eip:0x001000bb args:0x00000000 0xffff0000 0x00007b84 0x00000029
    kern/init/init.c:53: grade_backtrace1+38
ebp:0x00007b78 eip:0x001000d9 args:0x00000000 0x00100000 0xffff0000 0x0000001d
    kern/init/init.c:58: grade_backtrace0+23
ebp:0x00007b98 eip:0x001000fe args:0x001032fc 0x001032e0 0x0000130a 0x00000000
    kern/init/init.c:63: grade_backtrace+34
ebp:0x00007bc8 eip:0x00100055 args:0x00000000 0x00000000 0x00000000 0x00010094
    kern/init/init.c:28: kern_init+84
ebp:0x00007bf8 eip:0x00007d68 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
    <unknow>: -- 0x00007d67 --
++ setup timer interrupts
```

* 关注最后一行（终止处）

  > bootasm.S ---> call `bootmain()`  : bootloader ---> call `kern_init()`  : OS kernel
  >
  >  在`bootmain.c`中
  >
  > ```C
  >     // call the entry point from the ELF header
  >     // note: does not return
  >     ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
  > ```
  >
  > 也就是调用`kern_init`

  ```
  --- stack frame graph ---
  
    -----------------
   |                 | 0x7c00 (stack top)
    -----------------
   |     ret add     | 0x7bfc : [old EIP]           
    -----------------
   |     old ebp     | 0x7bf0 <--- [ebp]
    -----------------
   |                 | 
  ```

  

  * `ebp:0x00007bf8`

    * 此时ebp的值是`kern_init`函数的栈顶地址，从`obj/bootblock.asm`文件中知道整个栈的栈顶地址为0x00007c00，ebp指向的栈位置存放`调用者的ebp寄存器`的值，ebp+4指向的栈位置存放返回地址的值，这意味着kern_init函数的调用者（也就是bootmain函数）没有传递任何输入参数给它！因为单是存放旧的ebp、返回地址已经占用8字节了。

  * ` eip:0x00007d68` 

    * eip的值是`kern_init`函数的`返回地址`，也就是`bootmain`函数调用`kern_init`对应的指令的下一条指令的地址。这与`obj/bootblock.asm`是相符合的。

      ```assembly
      (gdb) x /10i 0x00007d66
         0x7d66:	call   *%eax
         0x7d68:	mov    $0xffff8a00,%eax
      ```

      注意从不同处截取查看指令是不同的（解读不同）

  * ` args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8`

    * 一般来说，args存放的4个dword是对应4个输入参数的值。但这里比较特殊，由于`bootmain`函数调用kern_init并没传递任何输入参数，并且栈顶的位置恰好在boot loader第一条指令存放的地址的上面，而args恰好是kern_int的ebp寄存器指向的栈顶往上第2~5个单元，因此args存放的就是boot loader指令的前16个字节！可以对比`obj/bootblock.asm`文件来验证（验证时要注意系统是小端字节序）。

      ```assembly
      00007c00 <start>:
          7c00:	fa                   	cli    
          7c01:	fc                   	cld    
          7c02:	31 c0                	xor    %eax,%eax
          7c04:	8e d8                	mov    %eax,%ds
          7c06:	8e c0                	mov    %eax,%es
          7c08:	8e d0                	mov    %eax,%ss
          7c0a:	e4 64                	in     $0x64,%al
          7c0c:	a8 02                	test   $0x2,%al
          7c0e:	75 fa                	jne    7c0a <seta20.1>
      ```

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



### 中断类型分类

* 异步中断 `asynchronous interrupt`

  > 也称外部中断,简称中断`interrupt`
  >
  > **由CPU外部设备引起的外部事件如I/O中断、时钟中断、控制台中断等是异步产生的**（即产生的时刻不确定），与CPU的执行无关。

* 同步中断 `synchronous interrupt`

  > 也称内部中断，简称异常`exception`
  >
  > 而把**在CPU执行指令期间检测到不正常的或非法的条件**(如除零错、地址访问越界)所引起的内部事件称作同步中断

* 陷入中断`trap interrupt`

  > 也称软中断`soft interrupt`,系统调用`system call`简称`trap`
  >
  > 把在**程序中使用请求系统服务的系统调用**而引发的事件，称作陷入中断

### 中断处理概述

> 1. 本实验只描述**保护模式**下的处理过程
> 2. 当CPU收到中断（通过`8259A`完成，有关8259A的信息请看附录A）或者异常的事件时，它会暂停执行当前的程序或任务，通过一定的机制跳转到负责处理这个信号的相关处理例程中，在完成对这个事件的处理后再跳回到刚才被打断的程序或任务中。
> 3. 中断向量和中断服务例程的对应关系主要是由`IDT`（中断描述符表）负责。操作系统在`IDT`中设置好各种中断向量对应的中断描述符，留待CPU在产生中断后查询对应中断服务例程的起始地址。而IDT本身的起始地址保存在`IDTR`寄存器中。

* 中断描述符表（Interrupt Descriptor Table`idt`） 

  * 中断描述符表把每个中断或异常编号和一个指向中断服务例程的描述符联系起来。同GDT一样，IDT是一个8字节的描述符数组，但IDT的第一项可以包含一个描述符。CPU把中断（异常）号乘以8做为IDT的索引。IDT可以位于内存的任意位置，CPU通过IDT寄存器（IDTR）的内容来寻址IDT的起始地址。指令LIDT和SIDT用来操作IDTR。两条指令都有一个显示的操作数：一个6字节表示的内存地址。指令的含义如下：

    - LIDT（Load IDT Register）指令：使用一个包含线性地址基址和界限的内存操作数来加载IDT。操作系统创建IDT时需要执行它来设定IDT的起始地址。这条指令只能在特权级0执行。（可参见libs/x86.h中的lidt函数实现，其实就是一条汇编指令）

    - SIDT（Store IDT Register）指令：拷贝IDTR的基址和界限部分到一个内存地址。这条指令可以在任意特权级执行。

  * 在保护模式下，最多会存在256个Interrupt/Exception Vectors。

    * 范围[0，31]内的32个向量被异常Exception和NMI使用，但当前并非所有这32个向量都已经被使用，有几个当前没有被使用的，请不要擅自使用它们，它们被保留，以备将来可能增加新的Exception。
    * 范围[32，255]内的向量被保留给用户定义的Interrupts。Intel没有定义，也没有保留这些Interrupts。用户可以将它们用作**外部I/O设备中断**`8259A IRQ`或者**系统调用**(`System Call `、`Software Interrupts`等)。

* IDT gate descriptors
  * 可参见`kern/mm/mmu.h`中的`struct gatedesc`数据结构对中断描述符的具体定义。
  * 在IDT中，可以包含如下3种类型的Descriptor：
    - Task-gate descriptor （这里没有使用）
    - Interrupt-gate descriptor （中断方式用到）
    - Trap-gate descriptor（系统调用用到）
  * 用户进程在正常执行中是不能禁止中断的，而当它发出系统调用后，将通过Trap Gate完成了从用户态（ring 3）的用户进程进了核心态（ring 0）的OS kernel。

### 中断处理硬件的工作

> 中断服务例程包括具体负责处理中断（异常）的代码是操作系统的重要组成部分。需要注意区别的是，有两个过程由硬件完成：

总处理过程示意图

```
Notes:
> summary : 中断向量 
            --IDT-> 中断描述符:1.[16~31]bit 段选择子(->GDT) 
                              2.[0~15]bit, [48~63]bit 构成的偏移地址
            --GDT-> 段描述符:segment base (段基址) 
            --base:offset-> ISA
            --ISA-> handle with interrupt
            
# Process : exception/interrupt(interrupt vector) ---> IDT ---> GDT ---> ISR
# IDT : Interrupt Descriptor Table 中断描述符表
# GDT : Global Descriptor Table 全局描述符表
# ISR : Interrupt Service Routine 中断服务程序
# CPL : Current Privilege Level 当前特权级(当前活动代码段的特权级)
# DPL : Descriptor Privilege Level 描述符特权(段本身真正的特权级)

CPU execute per instruction
	+ --- check `interrupt controller(8259A)`
		+ --- interrupt occurs
			+ beginning
			|
			| CPU read interrupt vector from bus
			| according to interrupt request
			|
			+ interrupt vector (as index)
			|
			| IDT
			|
			+ interrupt descriptor (include `segment selector`)
			|
			| GDT
			|
			+ segment descriptor (include segment base and other information)
			|
			| CPU get starting address of ISA
			|
			+ ISA
			|
			| 判断是否发生特权级转换 (according to CPL and DPL)
			| if so, save ss and esp
			|
			+ ***Save*** the `interrupted program` scene (eflags, cs, eip, ...) ----+
			|                                                                       |
			| CPU load CS:IP from `segment descriptor`                              |
			|                                                                       |
			+ executing ISA                                                         |
			|                                                                       |
			| ...                                                                   |
			|                                                                       |
			+ iret (see details below)                                              |
			|                                                                       |
			|                                                                       |
			|                                                                       |
			+ ***Resume*** the `interrupted program` scene (eflags, cs, eip, ...) --+
			
	*** Attention ***
			+ iret
				+ 1.
				| Resume ...
				+ 2.
				| 判断是否发生特权级转换 (according to CPL and DPL)
			  | if so, resume ss and esp
			  + 3.(due to ISA)
			  | popup the errorCode
			  +
```

1. 起始

   硬件中断处理过程1（起始）：从CPU收到中断事件后，打断当前程序或任务的执行，根据某种机制跳转到中断服务例程去执行的过程。

   ![interrupt](D:\github\OS\thu\images\interrupt.png)

   - CPU在**执行完当前程序的每一条指令**后，都会去确认在执行刚才的指令过程中中断控制器（如：`8259A`）是否发送中断请求过来，如果有那么CPU就会在相应的时钟脉冲到来时从总线上读取中断请求对应的中断向量；
   - CPU根据得到的中断向量（以此为索引）到IDT中找到该向量对应的中断描述符，中断描述符里保存着中断服务例程的段选择子；
   - CPU使用IDT查到的中断服务例程的段选择子从GDT中取得相应的段描述符，段描述符里保存了中断服务例程的段基址和属性信息，此时CPU就得到了中断服务例程的起始地址，并跳转到该地址；
   - CPU会根据CPL和中断服务例程的段描述符的DPL信息确认**是否发生了特权级的转换**。比如当前程序正运行在用户态，而中断程序是运行在内核态的，则意味着发生了特权级的转换，这时CPU会从当前程序的TSS信息（该信息在内存中的起始地址存在TR寄存器中）里取得该程序的内核栈地址，即包括内核态的ss和esp的值，并立即将系统当前使用的栈切换成新的内核栈。这个栈就是即将运行的中断服务程序要使用的栈。紧接着就将当前程序使用的用户态的ss和esp压到新的内核栈中保存起来；
   - CPU需要开始保存当前被打断的程序的现场（即一些寄存器的值），以便于将来恢复被打断的程序继续执行。这需要利用内核栈来保存相关现场信息，即依次压入当前被打断程序使用的eflags，cs，eip，errorCode（如果是有错误码的异常）信息；
   - CPU利用中断服务例程的段描述符将其第一条指令的地址加载到cs和eip寄存器中，开始执行中断服务例程。这意味着先前的程序被暂停执行，中断服务程序正式开始工作。

2. 结束

   硬件中断处理过程2（结束）：每个中断服务例程在有中断处理工作完成后需要通过iret（或iretd）指令恢复被打断的程序的执行。CPU执行IRET指令的具体过程如下：

   - 程序执行这条iret指令时，首先会从内核栈里弹出先前保存的被打断的程序的现场信息，即eflags，cs，eip重新开始执行；

   - 如果存在特权级转换（从内核态转换到用户态），则还需要从内核栈中弹出用户态栈的ss和esp，这样也意味着栈也被切换回原先使用的用户态的栈了；

   - **如果此次处理的是带有错误码（errorCode）的异常，CPU在恢复先前程序的现场时，并不会弹出errorCode。这一步需要通过软件完成，即要求相关的中断服务例程在调用iret返回之前添加出栈代码主动弹出errorCode。**

   - 堆栈切换（通过`TSS`完成）

     ```
      
      *** Stack Usage with Privilege-Level Change ***
     
      Interrupted Procedure's Stack
      +-----------------+
      |                 |        
      +-----------------+
      |                 | <--- ESP Before Transfer to Handler
      +-----------------+
      |                 | 
      +-----------------+
      |                 | 
      +-----------------+
      |                 |
      
      
      Handler's Stack
      +-----------------+
      |       SS        |        
      +-----------------+
      |       ESP       | 
      +-----------------+
      |     EFLAGS      | 
      +-----------------+
      |       CS        | 
      +-----------------+
      |       EIP       |
      +-----------------+
      |    Error Code   | <--- ESP After Transfer to Handler
     ```

### 中断特权级转换

> 中断处理得特权级转换是通过门描述符（gate descriptor）和相关指令来完成的。

1. 一个门描述符就是一个系统类型的段描述符，一共有4个子类型：
   1. 调用门描述符（call-gate descriptor）
   2. 中断门描述符（interrupt-gate descriptor）
   3. 陷阱门描述符（trap-gate descriptor）
   4. 任务门描述符（task-gate descriptor）
2. 与中断处理相关的是**中断门描述符**和**陷阱门描述符**。
3. 这些门描述符被存储在中断描述符表（Interrupt Descriptor Table，简称IDT）当中。
4. CPU把中断向量作为IDT表项的索引，用来指出当中断发生时使用哪一个门描述符来处理中断。中断门描述符和陷阱门描述符几乎是一样的。

### 分析

* 查看内核初始化的函数`kern/init/init.c::kern_init(void)`

  ```C
       16 int
       17 kern_init(void) {
       18     extern char edata[], end[];
       19     memset(edata, 0, end - edata);
       20 
       21     cons_init();                // init the console
       22 
       23     const char *message = "(THU.CST) os is loading ...";
       24     cprintf("%s\n\n", message);
       25 
       26     print_kerninfo();
       27 
       28     grade_backtrace();
       29 
       30     pmm_init();                 // init physical memory management
       31 
       32     pic_init();                 // init interrupt controller
       33     idt_init();                 // init interrupt descriptor table
       34 
       35     clock_init();               // init clock interrupt
       36     intr_enable();              // enable irq interrupt
       37 
       38     //LAB1: CAHLLENGE 1 If you try to do it, uncomment lab1_switch_test()
       39     // user/kernel mode switch test
       40     //lab1_switch_test();
       41 
       42     /* do nothing */
       43     while (1);
       44 }
  ```

  * `pic_init` : 中断控制器初始化

    * 在文件`kern/driver/picirq.c`中定义，用来初始化一个外设（特殊设备）即8259中断控制器
    * initialize the 8259A interrupt controllers
    * 把8259配置好之后，响应的外设就可以产生中断，并被CPU所接受和处理。

  * `idt_init` : 中断描述符表(中断向量表)`IDT`初始化

    * 在文件`kern/trap/trap.c`中定义

  * `clock_init` : 时钟中断

    * ucore 中定义每100 ticks产生一个clock interrupt

    * 关于`tick`

      > 系统时钟也称为时标或者Tick。 一个Tick的时长可以静态配置。 用户是以秒、毫秒为单位计时，而芯片CPU的计时是以Tick为单位的，当用户需要对系统操作时，例如任务挂起、延时等，输入秒为单位的数值，此时需要时间管理模块对二者进行转换。 Tick与秒之间的对应关系可以配置。

  * `intr_enable` : 使能中断

    * 源代码在`kern/driver/intr.c`

      ```C
            4 /* intr_enable - enable irq interrupt */
            5 void
            6 intr_enable(void) {
            7     sti();
            8 }
            9 
           10 /* intr_disable - disable irq interrupt */
           11 void
           12 intr_disable(void) {
           13     cli();
           14 }
      ```

      在`/libs/x86.h`中定义了`sti`和`cli`，其实是两条内联汇编，两条机器指令的含义是

      ```C
           72 static inline void
      ...skipping...
           78 sti(void) {
           79     asm volatile ("sti");
           80 }
           81 
           82 static inline void
           83 cli(void) {
           84     asm volatile ("cli");
           85 }
      ```

      * sti : 使能中断

      * cli : 屏蔽中断

        * 注意到从`0x7c00`运行时第一条指令是`cli`，可以在文件`bootasm.S`中查看

          ```assembly
               16     cli # Disable interrupts
               17     cld # String operations increment
          ```

        * 因此一开始`bootloader`启动时是处于中断屏蔽的状态

* 关于`trap.c`函数的实现背景分析

  > 中断服务的入口地址不是`trap`！

  ```C
      177 /* *
      178  * trap - handles or dispatches an exception/interrupt. if and when trap() returns,
      179  * the code in kern/trap/trapentry.S restores the old CPU state saved in the
      180  * trapframe and then uses the iret instruction to return from the exception.
      181  * */
      182 void
      183 trap(struct trapframe *tf) {
      184     // dispatch based on what type of trap occurred
      185     trap_dispatch(tf);
      186 }
  ```

  * 关于中断服务的入口地址，可见于`kern/trap/vectors.S`

    > 1. 定义了255个中断号对应的起始地址
    >
    > 2. vector.S 文件通过 vectors.c 自动生成，其中定义了每个中断的入口程序和入口地址 (保存在 vectors 数组中)。
    > 3. 中断可以分成两类：一类是压入错误编码的 (error code)，另一类不压入错误编码。对于第二类， vector.S 自动压入一个 0。（节选中包括了这两类）
    > 4. 此外，还会压入相应中断的中断号。在压入两个必要的参数之后，中断处理函数跳转到统一的入口 alltraps 处。
  
    节选
  
    ```assembly
         39 .globl vector7 
         40 vector7:
         41   pushl $0   <--- 不压入error code
         42   pushl $7
         43   jmp __alltraps
         
         44 .globl vector8  <--- 压入 error code
         45 vector8:
         46   pushl $8
         47   jmp __alltraps
    
    ```
  
    例如，当产生`98`号中断时，`ucore`产生的`IDT`（中断描述符表）会将CPU的PC指针（即`EIP`的值）指向
  
    ```assembly
    vector98:
    ```
  
    处并开始执行
  
    处理入口：`__alltraps`，定义见`trap/trapentry.S`
  
    ```assembly
          6 __alltraps:
          7     # push registers to build a trap frame
          8     # therefore make the stack look like a struct trapframe
          9     pushl %ds
         10     pushl %es
         11     pushl %fs
         12     pushl %gs
         13     pushal
         14 
         15     # load GD_KDATA into %ds and %es to set up data segments for kernel
         16     movl $GD_KDATA, %eax
         17     movw %ax, %ds
         18     movw %ax, %es
         19 
         20     # push %esp to pass a pointer to the trapframe as an argument to trap()
         21     pushl %esp
         22 
         23     # call trap(tf), where tf=%esp
         24     call trap  <------ 调用trap函数
         25 
         26     # pop the pushed stack pointer
         27     popl %esp
         28 
         29     # return falls through to trapret...
    ```
  
    产生中断后会保存现场，保存信息储存在`kernel`的中断栈`stack`中
  
  * `trap`会进一步调用`trap_dispatch`
  
    ```C
        137 /* trap_dispatch - dispatch based on what type of trap occurred */
        138 static void
        139 trap_dispatch(struct trapframe *tf) {
        140     char c;
        141 
        142     switch (tf->tf_trapno) {
        143     case IRQ_OFFSET + IRQ_TIMER:
        144         /* LAB1 YOUR CODE : STEP 3 */
        145         /* handle the timer interrupt */
        146         /* (1) After a timer interrupt, you should record this event using a global variable (increase it), such as ticks in k    146 ern/driver/clock.c
        147          * (2) Every TICK_NUM cycle, you can print some info using a funciton, such as print_ticks().
        148          * (3) Too Simple? Yes, I think so!
        149          */
        150         break;
        151     case IRQ_OFFSET + IRQ_COM1:
        152         c = cons_getc();
        153         cprintf("serial [%03d] %c\n", c, c);
        154         break;
        155     case IRQ_OFFSET + IRQ_KBD:
        156         c = cons_getc();
        157         cprintf("kbd [%03d] %c\n", c, c);
        158         break;
        159     //LAB1 CHALLENGE 1 : YOUR CODE you should modify below codes.
        160     case T_SWITCH_TOU:
        161     case T_SWITCH_TOK:
        162         panic("T_SWITCH_** ??\n");
        163         break;
        164     case IRQ_OFFSET + IRQ_IDE1:
        165     case IRQ_OFFSET + IRQ_IDE2:
        166         /* do nothing */
        167         break;
        168     default:
        169         // in kernel, it must be a mistake
        170         if ((tf->tf_cs & 3) == 0) {
        171             print_trapframe(tf);
        172             panic("unexpected trap in kernel.\n");
        173         }
        174     }
        175 }
    ```
  
    * 关于`IRQ_OFFSET`，`IRQ_TIMER`，`TICK_NUM`的定义
  
      file : `trap.h`
  
      ```C
           32 /* Hardware IRQ numbers. We receive these as (IRQ_OFFSET + IRQ_xx) */
           33 #define IRQ_OFFSET                32    // IRQ 0 corresponds to int IRQ_OFFSET
           35 #define IRQ_TIMER                0
      ```
  
      file : `trap.c`
  
      ```C
           12 #define TICK_NUM 100
      ```
  
    * 关于`trap_dispatch`中的参数类型`struct trap frame`
  
      ```C
           62 struct trapframe {
           63     struct pushregs tf_regs;
           64     uint16_t tf_gs;
           65     uint16_t tf_padding0;
           66     uint16_t tf_fs;
           67     uint16_t tf_padding1;
           68     uint16_t tf_es;
           69     uint16_t tf_padding2;
           70     uint16_t tf_ds;
           71     uint16_t tf_padding3;
           72     uint32_t tf_trapno;
           ++ -------------------------------------------------------------       
           73     /* below here defined by x86 hardware */
           74     uint32_t tf_err;
           75     uintptr_t tf_eip;
           76     uint16_t tf_cs;
           77     uint16_t tf_padding4;
           78     uint32_t tf_eflags;
                                                                硬件保存
           ++ -------------------------------------------------------------
                                                                软件保存
           79     /* below here only when crossing rings, such as from user to kernel */
           80     uintptr_t tf_esp;
           81     uint16_t tf_ss;
           82     uint16_t tf_padding5;
                  可能出现从用户产生中断要切换栈(stack)的情形(特权级转换)
           ++ -------------------------------------------------------------
           83 } __attribute__((packed));
      ```





### Q1

> 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

<img src="D:\github\OS\thu\images\lab1-6-IDT-and-IDTR.png" alt="lab1-6-IDT-and-IDTR" style="zoom:80%;" />

* 一个表项`8 bytes`

* 结构

  > 1. 其中第[16~31] 位是段选择子，用于索引全局描述符表GDT来获取中断处理代码对应的段地址。
  >
  > 2. 再加上第[0~15] 、[48~63]位构成的偏移地址，即可得到中断处理代码的入口。

  * bit 63 ... 48: offset 31..16
  * bit 47 ... 32: 属性信息，包括DPL、P flag等
  * bit 31 ... 16: Segment selector
  * bit 15 ... 0: offset 15..0


### Q2

> 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。

* idt_init函数的功能是初始化IDT表。IDT表中每个元素均为**门描述符**，记录一个中断向量的属性，包括中断向量对应的中断处理函数的段选择子/偏移量、门类型（是中断门还是陷阱门）、DPL等。因此，初始化IDT表实际上是初始化每个中断向量的这些属性。

* 题目已经提供中断向量的门类型和DPL的设置方法：

  > 除了系统调用的门类型为陷阱门、DPL=3外，其他中断的门类型均为中断门、DPL均为0.

* 





### Q3

> 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”。













