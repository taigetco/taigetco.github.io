---
layout:     post
title:      Volatile笔记整理
date:       2015-06-09 11:21:29
summary:    关于Java volatile使用和理解的一个总结
categories: java concurrency
---

其实网上已经由很多对java volatile的深入解析，之前也细读过，但始终觉得比较凌乱，也许是脑子理解问题．这里把自己所理解的东西写出来，当是对volatile的一个总结．

volatile变量可以atomic read和atomic write, 但多个steps合在一起时，不是线程安全，比如read/modify/write, 也就是说volatile_var = volatile_var + 1线程不安全．

在JVM中，对volatile的堆变量的写操作会添加一条汇编指令：
     0x01010101: lock addl $0x0,(%esp);
这条指令明显是一个空操作，相当于一个内存屏障，保证lock的前后指令不会重排序，且volatile变量对其他线程的可见性．

参考intel最新的Intel® 64 and IA-32 Architectures Software Developer’s Manual [5]

>For the Intel486 and Pentium processors, the LOCK# signal is always asserted on the bus during a LOCK operation, even if the area of memory being locked is cached in the processor.

>For the P6 and more recent processor families, if the area of memory being locked during a LOCK operation is cached in the processor that is performing the LOCK operation as write-back memory and is completely contained in a cache line, the processor may not assert the LOCK# signal on the bus. Instead, it will modify the memory location internally and allow it’s cache coherency mechanism to ensure that the operation is carried out atomically. This operation is called “cache locking.” The cache coherency mechanism automatically prevents two or more processors that have cached the same area of memory from simultaneously modifying data in that area.

所以在最新的处理器中，遇到LOCK指令，thread的cache会被更改，并且数据会被写回主存，这个操作一定是原子操作，那么如何使得其他处理器的缓存失效呢

>When operating in an MP system, IA-32 processors (beginning with the Intel486 processor) and Intel 64 processors have the ability to snoop other processor’s accesses to system memory and to their internal caches. They use this snooping ability to keep their internal caches consistent both with system memory and with the caches in other processors on the bus. For example, in the Pentium and P6 family processors, if through snooping one processor detects that another processor intends to write to a memory location that it currently has cached in shared state, the snooping processor will invalidate its cache line forcing it to perform a cache line fill the next time it accesses the same memory location.

>Beginning with the P6 family processors, if a processor detects (through snooping) that another processor is trying to access a memory location that it has modified in its cache, but has not yet written back to system memory, the snooping processor will signal the other processor (by means of the HITM# signal) that the cache line is held in modified state and will preform an implicit write-back of the modified data. The implicit write-back is transferred directly to the initial requesting processor and snooped by the memory controller to assure that system memory has been updated. Here, the processor with the valid data may pass the data to the other processors without actually writing it to system memory; however, it is the responsibility of the memory controller to snoop this operation and update memory.

在此再详细说下LOCK操作，LOCK操作的数据不在cache line或则数据大小操作一个cache line的时候，此时LOCK操作同样会锁住系统总线，此时这是个performance hit 操作，当然发生的概率比较低．Volatile中的lock是不会发生以上两种情况的．在最新的处理器中，更新主存已经不是通过简单的CPU写回策略进行的了，而是处理器之间直接传送有效数据，由内存管理器负责主存的更新。

[1] http://stackoverflow.com/questions/7805192/is-a-volatile-int-in-java-thread-safe
[2] http://codingcat.me/2015/05/09/big-brain-hole/
[3] http://www.infoq.com/cn/articles/ftf-java-volatile
[4] http://www.infoq.com/cn/articles/zzm-java-hsdis-jvm
[5] Intel® 64 and IA-32 Architectures Software Developer’s Manual
