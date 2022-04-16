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

  * 一把**排他性**的锁 : 对于锁对象`lk`

    * 在任何线程调度（线程执行的顺序）下
      * 若某`thread`持有`lk` ：lock(lk)返回且未释放
      * 那么任何`other threads` ：其lock(lk)都不能返回
    * p.s.状态机视角

* 





































