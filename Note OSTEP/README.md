# Operating Systems : Three Easy Pieces

> Notes or abstact of this book

[TOC]

## Introduction

### virtualizing the CPU

### virtualizing the memory

### concurrency

### persistence

### design goals

### some history

### summary

## Virtualization

### CPU scheduling

> 思想：
>
> 1. 运行最短工作，优化**周转**`turnaround time`时间
> 2. 交替运行所有工作，优化**响应**`response time`时间

#### introduction

* `SJF` : Shortest Job First
* `STCF` : Shortest Time-to-Completion First
* `RR` : Round Robin
  * 时间片`time slice`必须是时钟中断周期的倍数
  * 缩短响应时间，但周转时间很差
* `Incorporating I/O` : 重叠`overlap`

#### multi-level feedback queue `MLFQ`

* Basic rules :

  * **Rule 1** : if Priority(A)>Priority(B), A runs (B doesn't)
  * **Rule 2** : if Priority(A)=Priority(B), A&B run in RR (轮转)

* Attempt #1 : Change priority

  > By this way, `MLFQ` approximates `SJF`
  >
  > Disadvantages : 
  >
  > 1. starvation : some programs may game the scheduler
  > 2. behavior changing : i.e, what was `CPU-bound` may transition to a phase of `interactivity`(一个计算密集型的程序某段时间可能表现为一个交互型进程)

  * **Rule 3** : When a job enters the system, it is placed at **the highest priority.** `the topmost queue`
  * **Rule 4a** : If a job uses up an entire `time slice` while running, its **priority is reduced**. `i.e, it moves down one queue`（移入下一个队列）
  * **Rule 4b** : If a job gives up the CPU before the time slice is up, it stays at **the same priority** level.（针对`I/O`频繁程序）

* Attempt #2 : The Priority Boost

  > 周期性提升`boost`所有工作的优先级
  >
  > Disadvantage : 
  >
  > 常量的设置：What should S be set to?`voo-doo constant`,或者需要一些`black magic`

  * **Rule 5** : After some period S, move all the jobs in the system to the topmost queue.

* Attempt #3 : Better Accounting

  >优化**Rule 4**,防止资源被恶意程序（卡点`I/O`）垄断

  * **Rule 4** : Once a job uses up its entire time allotment at a given level(regardless of how many times it has given up the CPU), its priority is reduced (i.e., it moves down one queue).

* Summary `MLFQ`:

  > 使用该调度程序的系统：BSD UNIX, Windows NT, Windows series

  * Rule 1 : 优先级大先运行
  * Rule 2 : 优先级相等轮转运行
  * Rule 3 : 工作进入系统时置于最高优先级
  * Rule 4 : 用光某一层的时间配额（不论放弃多少次CPU），就降低优先级
  * Rule 5 : 经过一段时间S，将系统中所有工作重新加入最高优先级队列

  

#### Proportional Share

> 基于公平调度原则

* lottery scheduling

  * Ticket Mechanisms

    * ticket currency（彩票货币） : 每个用户通过自己的货币管理工作，但运行时根据比例兑换成全局货币

    * ticket transfer : 适用于 `client/server model`

    * ticket inflation : 适用于进程信任的环境

    * Implementation

      ```C
      int counter = 0;
      int winner = getrandom(0,totaltickets);
      note_t* current = head;
      while(current)
      {
        counter = counter + current->tickets;
        if(counter > winner)
          break;
        current = current->next;
      }
      //‘current’ is the winner: schedule it ...
      ```

* stride scheduling (步长调度)

  * 系统中每个工作都有自己步长，该步长`stride`与票数值成反比（用一个大数除以票数)

  * 每调度一次，该时间片`time slice`执行完后步长加在该工作的计数器中，用来描述**进度**`pass`

  * Implementation

    ```C
    current = remove_min(queue);		 //pick client with minimum pass
    schedule(current);						   //use resource for quantum
    current->pass += current->stride; //compute next pass using stride
    insert(queue,current);					  //put back into the queue
    ```

  * Disadvantage : needing global state

#### Multiprocessor Scheduling (Advanced)

> 先学一下并发的内容



## Memory scheduling





## Concurrency

## Persistence

