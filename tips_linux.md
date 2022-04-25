# Linux instructions

[TOC]

## 简记

### hexdump

> 以16进制查看文件

## gdb

> [Linux Tools Quick Tutorial](https://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/gdb.html?highlight=gdb)

### 隐藏功能

* `target record-full` 开始记录
  * 之后可以`watch $rax`去检测其值，有新旧值
* `target stop` 结束记录
* `reverse-step/reverse-stepi` 时间旅行调试(反向执行) 

### 运行

* `starti` : 在主函数入口处停下
  * 行不通时可以用两条指令代替：`break main` + `run`, 然后`layout asm` + `si`就可以单步调试了
* `si` : 单步运行
* `!` + `shell` : 加`!`可以调用外部系统指令

- run：简记为 r ，其作用是运行程序，当遇到断点后，程序会在断点处停止运行，等待用户输入下一步的命令。
- continue （简写c ）：继续执行，到下一个断点处（或运行结束）
- next：（简写 n），单步跟踪程序，当遇到函数调用时，也不进入此函数体；此命令同 step 的主要区别是，step 遇到用户自定义的函数，将步进到函数中去运行，而 next 则直接调用函数，不会进入到函数体内。
- step （简写s）：单步调试如果有函数调用，则进入函数；与命令n不同，n是不进入调用的函数的
- until：当你厌倦了在一个循环体内单步跟踪时，这个命令可以运行程序直到退出循环体。
- until+行号： 运行至某行，不仅仅用来跳出循环
- finish： 运行程序，直到当前函数完成返回，并打印函数返回时的堆栈地址和返回值及参数值等信息。
- call 函数(参数)：调用程序中可见的函数，并传递“参数”，如：call **gdb**_test(55)
- quit：简记为 q ，退出**gdb**

### 设置断点

- break n （简写b n）:在第n行处设置断点

  （可以带上代码路径和代码名称： b OAGUPDATE.cpp:578）

- b fn1 if a＞b：条件断点设置

- break func（break缩写为b）：在函数func()的入口处设置断点，如：break cb_button

- delete 断点号n：删除第n个断点

- disable 断点号n：暂停第n个断点

- enable 断点号n：开启第n个断点

- clear 行号n：清除第n行的断点

- info b （info breakpoints） ：显示当前程序的断点设置情况

- delete breakpoints：清除所有断点：

### 查看源代码

- list ：简记为 l ，其作用就是列出程序的源代码，默认每次显示10行。
- list 行号：将显示当前文件以“行号”为中心的前后10行代码，如：list 12
- list 函数名：将显示“函数名”所在函数的源代码，如：list main
- list ：不带参数，将接着上一次 list 命令的，输出下边的内容。
- 使用“disassemble /r”命令可以用16进制形式显示程序的原始机器码

### 打印表达式

- print 表达式：简记为 p ，其中“表达式”可以是任何当前正在被测试程序的有效表达式，比如当前正在调试C语言的程序，那么“表达式”可以是任何C语言的有效表达式，包括数字，变量甚至是函数调用。
- print a：将显示整数 a 的值
- print ++a：将把 a 中的值加1,并显示出来
- print name：将显示字符串 name 的值
- print **gdb**_test(22)：将以整数22作为参数调用 gdb_test() 函数
- print **gdb**_test(a)：将以变量 a 作为参数调用 gdb_test() 函数
- display 表达式：在单步运行时将非常有用，使用display命令设置一个表达式后，它将在每次单步进行指令后，紧接着输出被设置的表达式及值。如： display a
- watch 表达式：设置一个监视点，一旦被监视的“表达式”的值改变，**gdb**将强行终止正在被调试的程序。如： watch a
- whatis ：查询变量或函数
- info function： 查询函数
- 扩展info locals： 显示当前堆栈页的所有变量

### 查询运行信息

- where/bt ：当前运行的堆栈列表；
- bt backtrace 显示当前调用堆栈
- up/down 改变堆栈显示的深度
- set args 参数:指定运行时的参数
- show args：查看设置好的参数
- info program： 来查看程序的是否在运行，进程号，被暂停的原因。

### 分割窗口

- layout：用于分割窗口，可以一边查看代码，一边测试：
- layout src：显示源代码窗口
- `layout asm`：显示反汇编窗口
- layout regs：显示源代码/反汇编和CPU寄存器窗口
- layout split：显示源代码和反汇编窗口
- `ctrl `+ L：刷新窗口
- Ctrl + x，再按1：单窗口模式，显示一个窗口
- Ctrl + x，再按2：双窗口模式，显示两个窗口
- Ctrl + x，再按a：回到传统模式，即退出layout，回到执行layout之前的调试窗口。

## less

> less 与 more 类似，less 可以随意浏览文件，支持翻页和搜索，支持向上翻页和向下翻页。

### 参数

> man less

* `b` : 向上翻一页
* `d` : 向下翻半页
* `/` + 字符串：向下搜索"字符串"的功能
* `?` + 字符串：向上搜索"字符串"的功能
* `n` : 重复前一个搜索（与 / 或 ? 有关）

### 操作

1 全屏导航

- ctrl + F - 向前移动一屏
- ctrl + B - 向后移动一屏
- ctrl + D - 向前移动半屏
- ctrl + U - 向后移动半屏

2 单行导航

- j - 下一行
- k - 上一行

3 其它导航

- G - 移动到最后一行
- g - 移动到第一行
- q / ZZ - 退出 less 命令

4 其它有用的命令

- v - 使用配置的编辑器编辑当前文件
- h - 显示 less 的帮助文档
- &pattern - 仅显示匹配模式的行，而不是整个文件

### 实例

查看文件

```
less log2013.log
```

`ps`查看进程信息并通过less分页显示

```
ps -ef | less
```

查看命令历史使用记录并通过less分页显示

```
[root@localhost test]# history | less
22  scp -r tomcat6.0.32 root@192.168.120.203:/opt/soft
23  cd ..
24  scp -r web root@192.168.120.203:/opt/
25  cd soft
26  ls
……省略……
```





















