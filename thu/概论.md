# 概论

* OS软件组成
  * Shell
  * GUI : WIMP (视窗Window, 图标Icon, 选单Menu, 指标Pointer)
  * Kernel

* OS四特征
  * 并发
  * 
  * 共享
  * 虚拟
  * 异步
* 操作系统家族
  * Unix BSD --->  freeBSD, IOS, hp, ...
  * Linux --->  redhat, ubuntu, debian, Android
  * Windows
* 操作系统演变
  * 单用户系统
  * 批处理系统
  * 多程序系统：多道系统
  * 分时
  * 个人计算机
  * 分布式计算：每个用户的多个系统，网络，多处理器... ...
* 工具
  * shell instructions
  * 系统维护工具：apt, git
  * 源码阅读和编辑工具：eclipse-CDT, understand, gedit, vim
  * 源码比较工具：diff, meld
  * 开发编译调试工具：gcc, gdb, make
  * 硬件模拟器：qemu
* x86-32硬件
  * Intel 80386 运行模式，内存架构，寄存器
  
  * 80386 是32-bit 处理器，即可以寻址的物理内存地址空间为2**32 = 4G bytes
  
  *  地址空间
  
    * 段机制启动，页机制未启动
      * 逻辑地址->段机制处理->线性地址==物理地址
    * 段机制，页机制均启动
      * 逻辑地址->段机制处理->线性地址->页机制处理->物理地址
  
  * 80386 寄存器
  
    * 通用寄存器
    * 段寄存器
    * 指令指针寄存器`EIP`
  
    * 标志寄存器`EFLAGS`
    * 控制寄存器
    * 系统地址寄存器，调试寄存器，测试寄存器





















