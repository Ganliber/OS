# Course : NJU

> 相关代码放在 pwn 库的 docker_env 的OS文件夹下
>
> 课程主页地址 ：[jyywiki](http://jyywiki.cn/OS/2022/)

[TOC]



## P3 system call (code)

C语言系统调用库视角实现"Hello world"

syscall函数调用`write`系统调用，向编号为1的文件描述符中写入地址为hello的缓冲区上长度为LENGTH的字节

```C
/* file name: system-demo1.c */
#include <unistd.h>
#include <sys/syscall.h>

#define LENGTH(arr) (sizeof(arr)/sizeof(arr[0]))

const char hello[] = "\033[01;31mHello, OS World\033[0m\n";
//try: const char * hello = ...

int main()
{
  syscall(SYS_write,1,hello,LENGTH(hello));
  syscall(SYS_exit,1);
}
```

编译即可运行

```shell
gcc syscall-demo.c
```

<img src="D:\github\OS\images\syscall-demo.png" alt="syscall-demo" style="zoom:50%;" />

查看syscall代码

首先对于编译阶段如果使用`-g`,就可以将源代码的信息包括进去（反汇编时可以同时查看源码）

```shell
gcc -g syscall-demo.c
```

查看汇编代码

```shell
objdump -S a.out
```

得到如下汇编代码（局部）

* \<syscall@plt>表明这个可执行文件是动态链接的

```assembly
int main()
{
1149:       f3 0f 1e fa             endbr64
114d:       55                      push   %rbp
114e:       48 89 e5                mov    %rsp,%rbp
syscall(SYS_write,1,hello,LENGTH(hello));
1151:       b9 1d 00 00 00          mov    $0x1d,%ecx
1156:       48 8d 15 b3 0e 00 00    lea    0xeb3(%rip),%rdx        # 2010 <hello>
115d:       be 01 00 00 00          mov    $0x1,%esi
1162:       bf 01 00 00 00          mov    $0x1,%edi
1167:       b8 00 00 00 00          mov    $0x0,%eax
116c:       e8 df fe ff ff          callq  1050 <syscall@plt>
syscall(SYS_exit,1);
1171:       be 01 00 00 00          mov    $0x1,%esi
1176:       bf 3c 00 00 00          mov    $0x3c,%edi
117b:       b8 00 00 00 00          mov    $0x0,%eax
1180:       e8 cb fe ff ff          callq  1050 <syscall@plt>
return 0;
1185:       b8 00 00 00 00          mov    $0x0,%eax
}
```

gdb 调试

> starti : 在程序执行第一条指令时就停下来
>
> info inferious : 查看当前进程的地址空间
>
> 在gdb中执行系统的命令：加`!`

执行`info inferious`查看当前进程的进程号`PID=137`

```shell
(gdb) info inferiors
Num  Description       Executable
* 1    process 137       /pwn/OS/a.out 
```

执行`! pmap 137` 打印进程被暂停时响应的地址空间

```assembly
(gdb) !pmap 137
137:   /pwn/OS/a.out
0000558f63c5e000      4K r---- a.out
0000558f63c5f000      4K r-x-- a.out
0000558f63c60000      4K r---- a.out
0000558f63c61000      8K rw--- a.out
00007f914ccd1000      4K r---- ld-2.31.so 
00007f914ccd2000    140K r-x-- ld-2.31.so
00007f914ccf5000     32K r---- ld-2.31.so
00007f914ccfe000      8K rw--- ld-2.31.so
00007f914cd00000      4K rw---   [ anon ]
00007ffccea93000    132K rw---   [ stack ]
00007ffccead0000     16K r----   [ anon ]
00007ffccead4000      4K r-x--   [ anon ]
total              360K    
```

? 查看main()之前发生了什么：

```tex
进程初始时到main()执行时，进程的内存中多了libc-2.31.so
	* /lib64/ld-linux-x86-64.so.2加载了libc
	* 之后`libc`完成了自己的初始化
```

> OS已经提前为我们加载好了`a.out`和最初始加载器`ld-2.31.so(即/lib64/ld-linux-x86-64.so.2)`, 会帮助我们加载`libc`

* `main()`的开始/结束并不是整个程序的开始/结束

```C
/* syscall-demo2.c */
#include <stdio.h>

__attribute__((constructor)) void hello(){
  printf("Hello World!\n");
}

__attribute__((destructor)) void goodbye(){
  printf("Goodbye, Crul OS World!\n");
}

int main(){}
```

该程序在`main()`开始前打印`Hello World!`，在`main()`结束后打印`Coodbye, Curl OS World!`

### Trace

> OS课中很重要的一个工具 : strace

* strace - trace system calls and signals
* 基于ptrace ( process trace ) 实现
* 打印一个进程的系统调用系列，并且ta可以在系统调用之前就把进程拦截下来

示例

```shell
gcc -g syscall-demo2.c  # 生成a.out可执行文件
```

查看进程调用系列

```shell
strace ./a.out
--- 得到一系列的系统调:这些syscall都是加载器和libc调用的
execve(...)
brk(NULL)
arch_prctl(...)
...
```

注意到strace在调用write指令的时候同时将该系统调用"执行"了,也就是响应的内容打印到了screen上面

```
write(1, "Hello World!\n", 13Hello World!
)          = 13
write(1, "Goodbye, Crul OS World!\n", 24Goodbye, Crul OS World!
) = 24 
```

可以重定向到文件`/dev/null`，所有写入的数据将被丢弃

```shell
strace ./a.out > /dev/null
```

这样就得到了干净（没有标准输出）的strace结果输出

```
write(1, "Hello World!\nGoodbye, Crul OS Wo"..., 37) = 37 
```

### Summary

对于应用程序的加载执行过程：

* 被OS加载
  * 通过父进程的 `execve`
* 不断执行系统调用
  * 进程管理：fork（创建进程）, execve（执行某个程序）, exit, ...
  * 进程间通信：send, recv
  * 文件/设备管理：open, close, read, write
  * 储存管理：mmap（申请内存）, brk
* 直到 _exit (exit_group (退出进程的所有线程)) 退出

编译器`gcc`

* strace -f gcc a.c (gcc 会启动其他进程)
* 主要的系统调用
  * cc1 - 汇编器 (`.c` -> `.asm`)
  * as - 汇编器 (`.asm` -> ELF relocatable)
  * collect2 - 收集器 (收集构造函数的信息)
  * ld - 链接

图形界面程序`xedit (基于X-Window的窗口管理系统的编辑器)`, 本质上和其他任何图形界面程序(如vscode)

跟踪系统调用

* strace xedit
  * 主要的系统调用 : poll, recvmsg, writev
  * 图形界面程序和`X-Window`服务器按照`X11`协议通信
  * 虚拟机中的`xedit`将`X11`命令通过`ssh (X11 forwarding)`转发到`Host`

API示例

* 窗口管理器：设备屏幕管理 (read/write/mmap), 进程间通信(send, recv)
* 任务管理器：访问操作系统提供的进程对象 (readdir/read)
* 杀毒软件：文件静态扫描(read), 动态防御(ptrace)



## P4 多处理器编程

### Concurrency

#### 并发

* 并发性的来源：进程会调用操作系统的API

* 典型并发系统

  * 并发`Concurrent` : 多个执行流可以不按照一个特定的顺序执行

  * 并行`Parallel` : 允许多个执行流同时执行（多个处理器）

    | 处理器   | 共享内存? | 典型并发系统                        | 并发`C`/并行`P`    |
    | -------- | --------- | ----------------------------------- | ------------------ |
    | 单处理器 | Y         | OS kernel / 多线程程序              | C &&~P(并发不并行) |
    | 多处理器 | Y         | OS kernel / 多线程程序 / GPU kernel | C && P             |
    | 多处理器 | N         | 分布式系统（消息通信）              | C && P             |

#### 多处理器编程(多线程编程)

##### 线程

* 线程：多个执行流并发/并行执行，并且**共享内存**

  * 两个执行流共享：代码，所有全局变量（数据，堆区）
  * 线程之间指令的执行顺序是不确定`non-deterministic`的

  * 线程应共享什么？不应共享什么？

    栈和寄存器`独享` 

    ```C
    extern int x;
    int foo()
    {
      int volatile t = x;
      t += 1;
      x = t;
    }
    ```

    * foo的代码是 `共享`
    * 寄存器：rip, rsp, rax `独享`

* 关于`volatile`的解释
  * 详情见 ：[volatile cainiao](https://www.runoob.com/w3cnote/c-volatile-keyword.html)
  * 英文解释：*A volatile specifier is a hint to a compiler that an object may change its value in ways not specified by the language so that `aggressive optimizations` must be avoided.*
  * `volatile` 关键字是一种类型修饰符，用它声明的类型变量表示可以被某些编译器未知的因素更改，比如：操作系统、硬件或者其它线程等。遇到这个关键字声明的变量，**编译器对访问该变量的代码就不再进行优化，从而可以提供对特殊地址的稳定访问**。声明时语法：**int volatile vInt;** 当要求使用 volatile 声明的变量的值的时候，系统总是重新从它所在的内存读取数据，即使它前面的指令刚刚从该处读取过数据。而且读取的数据立刻被保存。
  * 用途：
    *  中断服务程序中修改的供其它程序检测的变量需要加 volatile；
    * 多任务环境下各任务间共享的标志应该加 volatile；
    * 存储器映射的硬件寄存器通常也要加 volatile 说明，因为每次对它的读写都可能由不同意义；

* `POSIX Threads`

  * POSIX为我们提供了线程库`pthreads`

    * `pthread_create` : 创建并运行线程
    * `pthread_join` : 等待某个线程结束
    * `man 7 pthreads` : 查看pthreads手册

  * 该课程封装了线程API`threads.h`

    ```C++
    #include <stdlib.h>
    #include <stdio.h>
    #include <string.h>
    #include <stdatomic.h>
    #include <assert.h>
    #include <unistd.h>
    #include <pthread.h>
    
    #define NTHREAD 64
    enum { T_FREE = 0, T_LIVE, T_DEAD, };
    struct thread {
      int id, status;
      pthread_t thread;
      void (*entry)(int);
    };
    
    struct thread tpool[NTHREAD], *tptr = tpool;
    
    void *wrapper(void *arg) {
      struct thread *thread = (struct thread *)arg;
      thread->entry(thread->id);
      return NULL;
    }
    
    void create(void *fn) {
      assert(tptr - tpool < NTHREAD);
      *tptr = (struct thread) {
        .id = tptr - tpool + 1,
        .status = T_LIVE,
        .entry = fn,
      };
      pthread_create(&(tptr->thread), NULL, wrapper, tptr);
      ++tptr;
    }
    
    void join() {
      for (int i = 0; i < NTHREAD; i++) {
        struct thread *t = &tpool[i];
        if (t->status == T_LIVE) {
          pthread_join(t->thread, NULL);
          t->status = T_DEAD;
        }
      }
    }
    
    __attribute__((destructor)) void cleanup() {
      join();
    }
    ```

    注解：

    * `create(fn)	p.s.fn is a function pointer`

      * 创建并运行一个线程，立即执行函数`fn`
      * 函数原型`void fn(int tid){}`
      * tid从1开始编号

    * `join(fn)`

      * 等待所有线程执行结束
      * 执行函数fn
      * 假定只能`join`一次

    * 数据结构(由于注释颜色较浅就用---代替了)

      ```C
      struct thread {
        int id; --- 线程号
        pthread_t thread; --- pthread 线程, pthread_t是POSIX提供的`线程号`类型
        void (*entry)(int); --- 入口地址
        struct thread *next; --- 链表
      }
      struct thread *threads; ---链表头
      ```

      















