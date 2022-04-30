# Lab2-1

> 物理内存管理

[TOC]

# 知识准备



## x86 特权级

> `Privilege levels`

* 特权级

  ```
   &1 Protection Rings
  
                 *
              *    *
           *    *    *
       *    *    *    *
       0    1    2    3
       |    |    |    |
       |    |    |    +----> Applications
       |    +----+---------> Operating System Services(Device Drivers,etc)
       +-------------------> Operating System Kernel
       
   &2 Privilege Levels
       
  Highest                                    Lowest
     |             |             |             |
     +-------------+-------------+-------------+
     
     * uCore and Linux only use ring0 and ring3
  ```

  

* 一些指令（特权指令）只能执行在`ring 0`

* CPU 检查特权级

  * 访问数据段
  * 访问页
  * 进入中断服务例程`ISRs`

  * ...

* 检查失败会产生：`General Protection Fault!`

### 段选择子与段/门描述符

* 当前特权级

  > 两位：00,01,10,11

  段选择子`Segment Selector`

  ```
  +-----------------+---------+
  |                 | RPL/CPL |
  +-----------------+---------+
  15                1         0
  * 处于最低两位`2 bit`
  ```

  * RPL `Requested Privilege Level`
    * 对应数据段，当前需要访问数据段对应的特权级
    * 存在于段寄存器：DS，ES，FS，GS
  * CPL `Current Privilege Level`
    * 对应代码段，当前代码段特权级
    * 存在于段寄存器：CS，SS

  段/门描述符

  ```
  +-----------...---+-----+---...---+
  |                 | DPL |         |
  +-----------...---+-----+---...---+
  31                14    13        0
  ```

  * DPL `Descriptor Privilege Level`
    * 当前需要访问的段（不论是代码段还是数据段），代表访问目标特权级
    * 段描述符，门描述符

* 访问条件

  * 访问门`Gate`时：`CPL <= DPL[ Gate ] & CPL >= DPL[ Segment ]`
    * 针对`Interrupt`, `Trap`, `Exception`,即门情况
    * 代码特权级要比门高，同时要访问处于更高特权级（内核态）
  * 访问段`Segment`时：`MAX( CPL, RPL ) <= DPL[ Segment ]`



### 通过中断切换特权级

> 特权级切换方法还有很多，中断只是其中之一

* 流程（以ring0 ---> ring 3为例)

  模拟栈 ---> 修改：SS( RPL = 3 ), CS( CPL = 3 ) ; 添加 ESP ---> iret指令弹出，然后就可以实现特权级转换

* 







## x86 内存管理单元 

> `MMU ( Memory Management Unit )`





















# 代码

