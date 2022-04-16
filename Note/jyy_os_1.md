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



## P4 多处理器编程 ：入门到放弃

### 总结

* 不要自作主张写”聪明“的并发程序
* 老老实实使用久经考验的API

### 入门

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

* `POSIX Threads`

  * POSIX为我们提供了线程库`pthreads`

    * `pthread_create` : 创建并运行线程
    * `pthread_join` : 等待某个线程结束
    * `man 7 pthreads` : 查看pthreads手册

  * 该课程封装了线程API`threads.h`

    最新版本(不过好像不能跑)`thread_latest.h`
    
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

    视频版本

    ```C
    #include <stdlib.h>
    #include <stdio.h>
    #include <string.h>
    #include <stdatomic.h>
    #include <assert.h>
    #include <unistd.h>
    #include <pthread.h>
    
    enum { T_FREE = 0, T_LIVE, T_DEAD, };
    struct thread {
      int id;
      pthread_t thread;
      void (*entry)(int);
      struct thread *next;
    };
    
    struct thread *threads; //Header of list
    
    void (*join_fn)(); //callback
    
    // =============== Basics =================== //
    
    __attribute__((destructor)) static void join_all(){
      for(struct thread *next;threads;threads=next)
      {
        pthread_join(threads->thread,NULL);
        next = threads->next;
        free(threads);
      }
      join_fn ? join_fn() : (void)0;
    }
    
    static inline void *entry_all(void *arg)
    {
      struct thread *thread = (struct thread *)arg;
      thread->entry(thread->id);
      return NULL;
    }
    
    static inline void create(void *fn)
    {
      struct thread *cur = (struct thread *)malloc(sizeof(struct thread));
      assert(cur);
      cur->id = threads ? threads->id + 1 : 1;
      cur->next = threads;
      cur->entry = (void (*)(int))fn;
      threads = cur;
      pthread_create(&cur->thread, NULL, entry_all, cur);
    }
    
    static inline void join(void (*fn)())
    {
      join_fn = fn;
    }
    ```
    
    
    
    注解（下方代码可能与最新版本的`threads.h`不太一样）：
    
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
        void (*entry)(int); --- 线程的入口地址
        struct thread *next; --- 下一个线程
      }; --- 分号不能少
      struct thread *threads; --- 链表头
      void (*join_fn)(); --- callback
      ```
    
    * 线程的创建
    
      ```C++
      static inline void *entry_all(void *arg)
      {
        struct thread *thread = (struct thread *)arg;//取出线程对象的指针
        thread->entry(thread->id);
        return NULL;
      }
      static inline void create(void *fn)
      {
        ---分配新的线程对象
        struct thread *cur = (struct thread *)malloc(sizeof(struct thread));
        ---构建代码的robustness
        assert(cur);
        ---当线程链表为空时,该线程对象的id为1否则为上一个的+1
        cur->id = threads ? threads->id + 1 : 1;
        cur->next = threads;
        cur->entry = (void (*)(int))fn;
        threads = cur;
        pthread_create(&cur->thread, NULL, entry_all, cur);
        /
         --参数3 : wrapper function
         --参数4 : 线程对象传进去(传到entry_all中)
        /
      }
      ```
      
      `pthread_create`
      
      ```C
      #include <pthread.h>
      int pthread_create(pthread_t *restrict thread,
      									const pthread_attr_t *restrict attr,
      									void *(*start_routine)(void *),
      									void *restrict arg);
      
      --- Compile and link with -pthread.
      ```
      
      直接使用`pthread_create(fn)`是不合法的，需要进行封装，因此有了`wrapper(包装) function`此处是`entry_all`
      
    * 线程`join`
    
      ```C
      static inline void join(void (*fn)())
      {
        join_fn = fn;/--- 函数指针赋值给全局变量join_fn
      }
      
      __attribute__((destructor)) static void join_all(){
        for(struct thread *next;threads;threads=next)
        {
          pthread_join(threads->thread,NULL);
          next = threads->next;
          free(threads);
        }
        join_fn ? join_fn() : (void) 0;
      }
      ```
    
    * 编译生成
    
      --- Compile and link with `-pthread`.

* 如何指导每个线程的堆栈范围和大小？`stack-probe.c`

  * 使用pmap可以查看到`8192 KiB`的内存映射区域和`4 KiB`(一页)的guard
  * 每个线程都要分配`8  MiB`的栈空间，但较多的线程不会耗尽内存

### 放弃

#### 总结

##### 放弃原因

* C代码
  * 编译器优化 --> 顺序的丧失 
* 处理器执行（二进制文件）
  * 中断 / 并行 --> 原子性(数据 独立性)的丧失
  * 处理器处理二进制文件时会选择**乱序执行** --> 可见性的丧失

##### 补救方法

* 保证顺序

  * 控制编译器行为，阻止编译优化

    > 此处告诉编译器变量`i`的所有读写都不能被优化

    ```C
    void delay(){
      for(volatile int i=0;i<DELAY_COUNT;i++)
      {...}
    }
    ```

  * 保证内存访问指令的顺序

    > 防止两个`x++`被合并，`y`不能被移到`barrier`之前

    ```C
    extern x;
    #define barrier() asm volatile ("":::"memory")
    
    void foo()
    {
      x++;
      barrier();
      x++; y++;
    }
    ```

* 保证原子性

  * 互斥 `mutual exclusion`

    * 效果等效于

      ```C
      stop_the_world(); 
      ... // critical section 临界区
      resume_the_world();
      ```

      执行`stop_the_world()` , 整个系统所有其他线程暂停

      执行 `resume_the_world()` , 系统中其他线程恢复

* 保证可见性
  * 基于`hardware`：通过指令集帮助实现
    * 顺序指令
      * fence (lfence, sfence, mfence, ...)
    * 原子指令
      * x86-family : lock prefix (lock xchg,...)
      * rsicv/mips : load-reservation/store-conditional(mip)

#### 放弃：原子性 `atomicity`

* 原子性
  - 单处理器多线程
    - 线程在运行时可能被中断，切换到另一个线程执行
  - 多处理器多线程
    - 线程根本就是并行执行的

* 多处理器的困难：共享资源

* 为了正确实现一个功能，需要原子性`atomicity`，历史上对`atomicity`有过研究热潮，但以失败告终（但几乎所有的实现都是错的，直到 [Dekker's Algorithm](https://en.wikipedia.org/wiki/Dekker's_algorithm)，还只能保证两个线程的互斥）。

#### 放弃：顺序

* 示例代码：`concur-demo1.c`在编译器不同优化下的结果

  <img src="D:\github\OS\images\concur-demo1-O012.png" alt="concur-demo1-O012" style="zoom:50%;" />

  运行得到结果如下

  <img src="D:\github\OS\images\result-O012.png" alt="result-O012" style="zoom:50%;" />

  不同的优化结果不同，具体查看其汇编代码

  
  
  * O1优化等价
  
    ```C
    t=sum;
    for(i=0;i<n;i++){}//空循环
    sum=t+n;
    ```
  
    执行顺序[1.1]->[1.2] wait ...->[2.1]->[2.2] wait ...->[1.3]->[2.3]
  
    ```
    P1
    --------------------------------------
    [1.1]t = sum                          
    [1.2]for(i=0;i<n;i++){} //空循环等待此时会跳转到P2
    [1.3]sum = t+n
    P2
    --------------------------------------
    [2.1]t=sum
    [2.2]for(i=0;i<n;i++){}
    [2.3]sum = t+n
    
    所初始取到的t都是0,之后进入空循环
    ```
  
  * O2优化等价
  
    ```C
    sum = sum+n
    ```
  
    由于高度优化，并发性可能性较低因此得到的结果是正确的
  
  ```assembly
  === O0 ===
  0000000000001376 <do_sum>:
  1376:       f3 0f 1e fa             endbr64
  137a:       55                      push   %rbp
  137b:       48 89 e5                mov    %rsp,%rbp
  137e:       c7 45 fc 00 00 00 00    movl   $0x0,-0x4(%rbp)
  1385:       eb 16                   jmp    139d <do_sum+0x27>
  1387:       48 8b 05 8a 2c 00 00    mov    0x2c8a(%rip),%rax        # 4018 <sum>
  138e:       48 83 c0 01             add    $0x1,%rax
  1392:       48 89 05 7f 2c 00 00    mov    %rax,0x2c7f(%rip)        # 4018 <sum>
  1399:       83 45 fc 01             addl   $0x1,-0x4(%rbp)
  139d:       81 7d fc ff e0 f5 05    cmpl   $0x5f5e0ff,-0x4(%rbp)
  13a4:       7e e1                   jle    1387 <do_sum+0x11>
  13a6:       90                      nop
  13a7:       90                      nop
  13a8:       5d                      pop    %rbp
  13a9:       c3                      retq
  
  === O1 ===
  0000000000001223 <do_sum>:
  1223:       f3 0f 1e fa             endbr64
  1227:       48 8b 15 ea 2d 00 00    mov    0x2dea(%rip),%rdx        # 4018 <sum>
  122e:       b8 00 e1 f5 05          mov    $0x5f5e100,%eax
  1233:       83 e8 01                sub    $0x1,%eax
  1236:       75 fb                   jne    1233 <do_sum+0x10>
  1238:       48 8d 82 00 e1 f5 05    lea    0x5f5e100(%rdx),%rax
  123f:       48 89 05 d2 2d 00 00    mov    %rax,0x2dd2(%rip)        # 4018 <sum>
  1246:       c3                      retq
  
  === O2 ===
  00000000000012e0 <do_sum>:
  12e0:       f3 0f 1e fa             endbr64
  12e4:       48 81 05 29 2d 00 00    addq   $0x5f5e100,0x2d29(%rip)        # 4018 <sum>
  12eb:       00 e1 f5 05
  12ef:       c3                      retq 
  ```
  
  

#### 放弃：可见性



### 本节知识补充

#### volatile

* 详情见 ：[volatile cainiao](https://www.runoob.com/w3cnote/c-volatile-keyword.html)
* 英文解释：*A volatile specifier is a hint to a compiler that an object may change its value in ways not specified by the language so that `aggressive optimizations` must be avoided.*
* `volatile` 关键字是一种类型修饰符，用它声明的类型变量表示可以被某些编译器未知的因素更改，比如：操作系统、硬件或者其它线程等。遇到这个关键字声明的变量，**编译器对访问该变量的代码就不再进行优化，从而可以提供对特殊地址的稳定访问**。声明时语法：**int volatile vInt;** 当要求使用 volatile 声明的变量的值的时候，系统总是重新从它所在的内存读取数据，即使它前面的指令刚刚从该处读取过数据。而且读取的数据立刻被保存。
* 用途：
  *  中断服务程序中修改的供其它程序检测的变量需要加 volatile；
  *  多任务环境下各任务间共享的标志应该加 volatile；
  *  存储器映射的硬件寄存器通常也要加 volatile 说明，因为每次对它的读写都可能由不同意义；

#### assert

注意：assert是宏，而不是函数。

assert`宏`的原型定义在<assert.h>中，其作用是如果它的条件返回错误，则终止程序执行，原型定义：

```C
#include <assert.h>
void assert( int expression );
```

assert的作用是现计算表达式 expression ：如果其值为假（即为0），那么它先向`stderr`打印一条出错信息，
然后通过调用 `abort` 来终止程序运行，否则，assert()无任何作用。

#### function pointer

解析`void (*f(int, void (*)(int)))(int)`

* 声明

  ```C
  void (*pfunc)(int); --- 函数指针pfunc指向的函数参数为int型,返回值为void型函数
  int (*p)(int i,int j); ---funtion pointer p 指向参数为两个int型,返回值为int型的函数 
  ```
  
* 赋值

  ```C++
  void func(int a){	a=5;}
  --- 两种方法都可以 ---
  pfunc = func;
  pfunc = &func;
  ```

  但有时候可能是一个常数`0x8999940`,它恰好也表示一个安全的与`function`相同的函数，如何将这个数值赋给*pfunc*呢？显然我们需要强制类型转换，应该将该常数转换成什么类型呢？这就是问题的关键！

  ```C++
  pfunc = 0x8999940;(×)
  ```

  在`void (*pfunc)(int)`语句里面，只有`pfunc`是变量名称，那么剩余的部分，`void(*)(int)`，即我们需要的转换类型

  ```C
  pfunc = (void (*)(int)) 0x8999940;
  ```

* 调用

  ```C++
  pfunc(5);
  (*pfunc)(5);
  ```

* 返回**函数指针**的函数声明

  需求分析

  1. 定义一个函数；

  2. 该函数特点 : 两参，返回值和其中一个参数均是函数指针，且均为`void (*)(int)`型
  3. 另一个参数为`int`型，函数名为`function`

  定义一

  ```C
  typedef void (*HANDLER)(int);
  HANDLER function(HANDLER,int);
  ```

  定义二

  ```C
  --- 若ret -> int, param1 -> int, param2 -> function pointer
    int function(int, void(*)(int));
  
  --- 对于`function(int, void(*)(int))`是一个int型(return type)数据,现将其改变为一个function pointer
  --- 根据function pointer定义
    ret_type (*fp_name)(param1,param2,...);
  	void (*fp_name)(int);
  
  --- fp_name 即 void (*)(int) 函数指针类型，视作 function(int, void(*)(int))
    void (* function(int, void(*)(int)))(int);
  
  --- 将function换成f即得开头的函数声明:
    void (*f(int, void (*)(int)))(int)
  ```

#### wrapper 函数

wrapper，中文意思为包装。 在计算机术语中，`wrapper 函数`指的是“主要用于调用其他函数的函数”。 在面向对象程序设计中，也被称作`method delegation`（方法委托）。

#### 关于gcc和cc

在Linux下一会看到cc，另一会又看到gcc，感觉又点混乱的样子。区别如下：

1. 首先，如果讨论范围在Unix和Linux之间，那么cc和gcc不是同一个东西。cc来自于Unix的c语言编译器，是 `c compiler` 的缩写。gcc来自Linux世界，是`GNU compiler collection` 的缩写，注意这是一个**编译器集合**，不仅仅是c或c++。
2. 其次， 如果讨论范围仅限于Linux，我们可以认为它们是一样的，在Linux下调用cc时，其实际上并不指向unix的cc编译器，而是指向了gcc，也就是说cc是gcc的一个链接（快捷方式），看看下面的终端输出就明白了：

```shell
nvidia@nvidia-desktop:~$ which cc
/usr/bin/cc
nvidia@nvidia-desktop:~$ ls -al /usr/bin/cc
lrwxrwxrwx 1 root root 20 6-р с 22  2018 /usr/bin/cc -> /etc/alternatives/cc
nvidia@nvidia-desktop:~$ ls -al /etc/alternatives/cc
lrwxrwxrwx 1 root root 12 6-р с 22  2018 /etc/alternatives/cc -> /usr/bin/gcc
```

为什么会这样，很简单，为了兼容性：
cc是Unix下的，是收费的，可不向Linux那样可以那来随便用，所以Linux下是没有cc的。然后，问题来了，如果我的c/c++项目是在Unix下编写的，在写makefile文件时自然地用了cc，当将其放到Linux下这无法make了，必须将其中的cc全部修改成gcc。这太麻烦了哈，所以，Linux这想了这么一个方便的解决方案：

> 不修改makefile，继续使用cc，这个cc是个“冒牌货”，它实际指向gcc。

#### Makefile

* 关于程序编译的一些规范和方法，一般来说，无论是C、C++、还是pas，首先要把源文件编译成中间代码文件，在Windows下也就是 .obj 文件，UNIX下是 .o 文件，即 Object File，这个动作叫做编译（compile）。然后再把大量的Object File合成执行文件，这个动作叫作链接（link）。

* 编译时，编译器需要的是语法的正确，函数与变量的声明的正确。对于后者，通常是你需要告诉编译器头文件的所在位置（头文件中应该只是声明，而定义应该放在C/C++文件中），只要所有的语法正确，编译器就可以编译出中间目标文件。一般来说，每个源文件都应该对应于一个中间目标文件（O文件或是OBJ文件）。

* 链接时，主要是链接函数和全局变量，所以，我们可以使用这些中间目标文件（O文件或是OBJ文件）来链接我们的应用程序。链接器并不管函数所在的源文件，只管函数的中间目标文件（Object File），在大多数时候，由于源文件太多，编译生成的中间目标文件太多，而在链接时需要明显地指出中间目标文件名，这对于编译很不方便，所以，我们要**给中间目标文件打个包**，在Windows下这种包叫“库文件”（Library File)，也就是 `.lib` 文件，在UNIX下，是Archive File，也就是 `.a` 文件。

* 格式

  ```makefile
  target ... : prerequisites ...
              command
              ...
              ...
  ```

  * `target`也就是一个目标文件，可以是Object File，也可以是执行文件。还可以是一个标签（Label），对于标签这种特性，在后续的“伪目标”章节中会有叙述。
  *  `prerequisites`就是，要生成那个target所需要的文件或是目标。
  * ` command`也就是make需要执行的命令。（任意的Shell命令）

* 示例

  > 一个有3个头文件，和8个C文件的工程文件的`makefile`编写

  ```makefile
  edit : main.o kbd.o command.o display.o /
             insert.o search.o files.o utils.o            <- 依赖文件
              cc -o edit main.o kbd.o command.o display.o /
                         insert.o search.o files.o utils.o
  
  main.o : main.c defs.h
              cc -c main.c
  kbd.o : kbd.c defs.h command.h
              cc -c kbd.c
  command.o : command.c defs.h command.h
              cc -c command.c
  display.o : display.c defs.h buffer.h
              cc -c display.c
  insert.o : insert.c defs.h buffer.h
              cc -c insert.c
  search.o : search.c defs.h buffer.h
              cc -c search.c
  files.o : files.c defs.h buffer.h command.h       <- 依赖文件
              cc -c files.c
  utils.o : utils.c defs.h
              cc -c utils.c
  clean :
  			rm edit main.o kbd.o command.o display.o /
  					insert.o search.o files.o utils.o
  ```

  * 注意每一行开头`gcc`需要`Tab`
  * `/`是换行符，便于`Makefile`的易读
  * 生成和删除
    * `make` : 在该目录下直接输入命令`make`就可以生成执行文件`edit`(上述代码的最终可执行文件为edit)
    * `make clean` : 删除执行文件和所有的中间目标文件
  * 在这个makefile中，目标文件（target）包含：执行文件`edit`和中间目标文件（*.o），依赖文件（prerequisites）就是冒号后面的那些 .c 文件和 .h文件。每一个 .o 文件都有一组依赖文件，而这些 .o 文件又是执行文件 edit 的依赖文件。依赖关系的实质上就是说明了目标文件是由哪些文件生成的，换言之，`目标文件是哪些文件更新(update)的`。
  * makefile不是简单的生成关系！其为智能化的版本更新，版本管理工具
  * 在定义好依赖关系后，后续的那一行定义了如何生成目标文件的操作系统命令，一定要以一个Tab键作为开头。记住，**make并不管命令是怎么工作的，他只管执行所定义的命令**。!!!!!!!!!!!!`make`会比较`targets`文件和`prerequisites`文件的`修改日期`，如果prerequisites文件的日期要比targets文件的日期要新，或者target不存在的话，那么，make就会执行后续定义的命令。
    









## P5 理解并发程序的执行

### 串行程序的状态机模型

> 单线程

#### 有限状态机 

> Finite State Machine (FSM)

* 有向图 G(F,E)

  * 节点 v ∈ V --> 状态
  * 边 e ∈ E --> 状态的转换

* 程序 <=> FSM

  * OS上程序执行时，状态有限

    * registers (including PC pointer)
    * memory : code(text), data, stack

  * 构造有限状态机

    * 每个不同的 configuration (mem/register) 都是状态机的节点
      * s = (M,R)∈V, 代表某个时刻程序mem/registers的快照`snapshot`
      * 16MiB内存有2\^(2\^24)种不同的状态
    * s = (M,R)的下一个状态是执行`M[R[%rip]]`处的指令得到的 s' = (M',R')
      * 取出PC指针处的指令、译码、执行、写回数据
      * **大部分**状态 s 有唯一的后续状态`deterministic`

  * 不确定`non-deterministic`的指令可能有**多个后续状态**，

    * (时间)`rdtsc/rdtscp`

      * 获取处理器的"时间戳"用于精确定时

    * (机器状态)`rdrand`

      * 处理器自身提供的"真"随机数指令 `since Ivy Bridge, 2013`

      * 例子`rdrand.c`，将`rdrand`的结果写到`val`中，将其返回结果打印到终端

        ```C
        #include <stdio.h>
        #include <stdint.h>
        int main()
        {
          uint64_t val;
          asm volatile ("rdrand %0": "=r"(val));
          printf("rdrand returns %016lx\n",val);
        } 
        ```

        每次打印的结果不一致

        <img src="D:\github\OS\images\rdrand.png" alt="rdrand" style="zoom:50%;" />

    * (系统调用)syscall -->更多的不确定性

      * 一般应用的不确定性来源
      * 例：`read(fd, buf, size)`
        * 返回值不确定(-1`read fail`, 0`file end`, 1`file bytes`, 2`file bytes`, ...)
        * `buf` 中的数据不确定(从键盘输入)
  
* `x86-64`举例

  * 16 通用寄存器
  * PC 指针 / 机器状态 /段寄存器(后五个)
    * rip, rflags, cs, ds, es, fs, gs

  * 内存
    * 操作系统分配：通过`procfs`查看

  * 这些状态都可以被`gdb`观察到
    * `info registers`
    * `info inferious`查看本进程进程号，通过`!cat /proc/[pid]/maps`查看进程的虚拟内存
      * `/proc/[pid]/maps` 有内存映射信息
      * `vvar`, `vdso`, `vsyscall`是系统映射的内存

* **"程序=状态机=有向图"** 是一种理解程序，分析程序的**工具**

  * 计算机科学领域的基础工具之一

  * 用`Graph`的思路理解`Program/Computer System` !

  * 状态机模型：应用

    * 计算机**硬件**上应用：高性能处理器

      * 超标量`superscalar`处理器
        * insight : 允许在状态机上跳跃(当状态机不分叉，即确定时，可以在一个时钟周期内合并多个本应逐步实现的指令实现处理器性能的提升，而状态机模型的思想是超标量处理器的设计哲学的来源之一)

    * 计算机**系统**上应用：程序分析技术

      * 静态分析：根据**程序代码**推导处状态机的性质
      * 动态分析：检查运行时检测到状态机的执行和转换
      * 实例1 ：`Time-Travel Debugging`
        * `gdb`隐藏功能
          * `target record-full` 开始记录
            * 之后可以`watch $rax`去检测其值，有新旧值
          * `target stop` 结束记录
          * `reverse-step/reverse-stepi` 时间旅行调试(反向执行) 


      * 实例2 ：`Record & Replay ` -->一种重要的调试工具
        * 记录 & 重放 ：在程序执行时记录信息，结束后重现程序的行为
        * 确定的程序不需要任何记录，记录的都是一些导致状态机不确定的行为，如`syscall`
        * 机制的实现：
          * 对于确定的结果：执行指令重放
          * 对于不确定的结果：通过将**记录的结果**替换（如当做返回值等等）实现重放
        * 只需要记录`non-deterministic`的指令
          * （单线程）应用程序
            * syscall, rdrand, rdtsc, ...
          * （单处理器）操作系统
            * mmio, in, out, rdrand, rdtsc, 中断, ...
            * `QEMU`内置了`Record/Replay`


### 并发程序的状态机模型

* 线程
  * 共享：代码段，堆，全局/静态变量
  * 独享：寄存器，栈
* 系统有`n`个线程，并发程序的状态：
  * s=(M, R1, R2, R3, ..., R4) （由于代码段，栈，数据段都在内存中，即同一块地址空间）
  * 并发系统执行指令的顺序`不确定`的
    * 每个(M, R1), (M, R2), ... , (M, Rn) 都可以堪称一个串行程序
    * 将`一个多线程的并发程序`看做`若干共享内存的串行程序`
    * 任意状态，可以选择任意一个线程执行一条指令，首先影响该 (M, Ri) 的状态进而影响整个(M, R1, R2, R3, ..., R4)的状态
    * 多线程程序的`nature`就是`non-deterministic`


### 理解并发程序的执行



#### 互斥

##### 例子

* 希望实现 `lock/unlock` ，保证

  * （顺序）编译优化不能越过 `lock/unlock`
  * （原子）`lock` 返回之后 `unlock` 返回之前，其他线程`lock`不得返回
  * （可见） `unlock` 之前执行的写操作，在`unlock`之后对其他线程可见

  ```C
  void do_sum()
  {
    for(int i=0;i<n;i++)
    {
      lock();//保证顺序，原子性，可见性
      /** critical section **/
      sum++;
      /**      临界区      **/
      unlock();
    }
  }
  ```

##### Peterson 算法 ：实现2人的互斥

> 只能满足但CPU双线程情况下的互斥

* 实例算法 : 想象独享标志旗子 (x, y) 和共享牌子 turn, 双重保障

  ```C
  --- 假设: 机器每次原子地(atomically)执行一次代码,内存访问立即可见 ---
  int ture = T1, x=0, y=0;
  void thread1()
  {
    [1] x=1;turn=T2;
    [2] while(y && turn==T2);// when y==0 || turn==T1, change to critical section
    [3] //critical section
    [4] x=0;
  }
  void thread2()
  {
    [1] y=1;turn=T1;
    [2] while(x && turn==T1);
    [3] //critical section
    [4] y=0;
  }
  ```

  * 构造FSM : 

    * State : (PC1, PC2, x, y, turn), 用上代码中的`[i]`表示PC，初始值 : (1,1,0,0,T1)

    * 执行演示：只有牌子和旗子**同时**是对方时自己才会死循环

      ```mermaid
      stateDiagram-v2
          (1,1,0,0,T1) --> (2,1,1,0,T2): T1
          %% 先进T1
          	(2,1,1,0,T2) --> (3,1,1,0,T2): T1
          		(3,1,1,0,T2) --> (4,1,1,0,T2): T1
          			(4,1,1,0,T2) --> (end,1,0,0,T2): T1
          				(end,1,0,0,T2) --> (end,2,0,1,T1): T2
          			(4,1,1,0,T2) --> (4,2,1,1,T1):
          		(3,1,1,0,T2) --> (3,2,1,1,T1): T2
      		%% 先进T2
          	(2,1,1,0,T2) --> (2,2,1,1,T1): T2
          		(2,2,1,1,T1) --> (3,2,1,1,T1): T1
          			(3,2,1,1,T1) --> (4,2,1,1,T1): T1
          				(4,2,1,1,T1) --> (end,2,0,1,T1): T1
          					(end,2,0,1,T1) --> (end,3,0,1,T1): T2
          					(end,3,0,1,T1) --> (end,4,0,1,T1): T2
          					(end,4,0,1,T1) --> (end,end,0,0,T1): T2
          				(4,2,1,1,T1) --> (4,2,1,1,T1): T2(Loop)
          			(3,2,1,1,T1) --> (3,2,1,1,T1): T2(Loop)
      				(2,2,1,1,T1) --> (2,2,1,1,T1): T2(Loop)
          	
          %%同理先执行T1的情况
          %%(1,1,0,0,T1) --> (1,2,0,1,T1): T2 
      ```

      需要满足的两点

      * `Safety` : PC1, PC2不能同时为[3]
      * `liveness` : 不可能在状态机有向图上不停地绕圈

      









