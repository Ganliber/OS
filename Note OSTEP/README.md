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

### scheduling

> 思想：
>
> 1. 运行最短工作，优化**周转**`turnaround time`时间
> 2. 交替运行所有工作，优化**响应**`response time`时间

##### introduction

* `SJF` : Shortest Job First
* `STCF` : Shortest Time-to-Completion First
* `RR` : Round Robin
  * 时间片`time slice`必须是时钟中断周期的倍数
  * 缩短响应时间，但周转时间很差
* `Incorporating I/O` : 重叠`overlap`

##### multi-level feedback queue `MLFQ`

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

  >

  * 

## Concurrency

## Persistence

