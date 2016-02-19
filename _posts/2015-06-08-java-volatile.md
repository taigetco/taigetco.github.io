##Volatile笔记整理

----------

其实网上已经由很多对java volatile的深入解析，之前也细读过，但始终觉得比较凌乱，也许是脑子理解问题．这里把自己所理解的东西写出来，当是对volatile的一个总结．

**Volatile在JMM里面定义两个语义**：
 - 不能重排
 - 保证data对所有线程可见

**非线程安全**
Volatile变量可以atomic read和atomic write, 但多个steps合在一起时，不是线程安全，比如read/modify/write, 也就是说volatile_var = volatile_var + 1线程不安全．

在JVM中，对volatile的堆变量的写操作会添加一条汇编指令，参考openJDK的x86_64.ad：
  `
    lock addl $0x0,(%esp);
   `
这条指令明显是一个空操作，相当于一个内存屏障，保证lock的前后指令不会重排序，且volatile变量对其他线程的可见性．

参考intel最新的Intel® 64 and IA-32 Architectures Software Developer’s Manual [5]

>For the Intel486 and Pentium processors, the LOCK# signal is always asserted on the bus during a LOCK operation, even if the area of memory being locked is cached in the processor.

>For the P6 and more recent processor families, if the area of memory being locked during a LOCK operation is cached in the processor that is performing the LOCK operation as write-back memory and is completely contained in a cache line, the processor may not assert the LOCK# signal on the bus. Instead, it will modify the memory location internally and allow it’s cache coherency mechanism to ensure that the operation is carried out atomically. This operation is called “cache locking.” The cache coherency mechanism automatically prevents two or more processors that have cached the same area of memory from simultaneously modifying data in that area.

所以在最新的处理器中，遇到LOCK指令，thread的cache会被更改，并且数据会被写回主存，这个操作一定是原子操作，那么如何使得其他处理器的缓存失效呢

>When operating in an MP system, IA-32 processors (beginning with the Intel486 processor) and Intel 64 processors have the ability to snoop other processor’s accesses to system memory and to their internal caches. They use this snooping ability to keep their internal caches consistent both with system memory and with the caches in other processors on the bus. For example, in the Pentium and P6 family processors, if through snooping one processor detects that another processor intends to write to a memory location that it currently has cached in shared state, the snooping processor will invalidate its cache line forcing it to perform a cache line fill the next time it accesses the same memory location.

>Beginning with the P6 family processors, if a processor detects (through snooping) that another processor is trying to access a memory location that it has modified in its cache, but has not yet written back to system memory, the snooping processor will signal the other processor (by means of the HITM# signal) that the cache line is held in modified state and will preform an implicit write-back of the modified data. The implicit write-back is transferred directly to the initial requesting processor and snooped by the memory controller to assure that system memory has been updated. Here, the processor with the valid data may pass the data to the other processors without actually writing it to system memory; however, it is the responsibility of the memory controller to snoop this operation and update memory.

在此再详细说下LOCK操作，LOCK操作的数据不在cache line或则数据大小操作一个cache line的时候，此时LOCK操作同样会锁住系统总线，此时这是个performance hit 操作，当然发生的概率比较低．Volatile中的lock是不会发生以上两种情况的．在最新的处理器中，更新主存已经不是通过简单的CPU写回策略进行的了，而是处理器之间直接传送有效数据，由内存管理器负责主存的更新。

**重排序**
有两种重排序，compiler和cpu. 一下只讨论compiler的重排序．下面这个例子定义６个field, 用PrintAssembly查看JIT生成的汇编: 
```java
public class TestJIT
{
    private static int field1;
    private static int field2;
    private static int field3;
    private static int field4;
    private static int field5;
    private static int field6;
     
    private static void assign(int i)
    {
        field1 = i << 1;
        field2 = i << 2;
        field3 = i << 3;
        field4 = i << 4;
        field5 = i << 5;
        field6 = i << 6;
    }
 
    public static void main(String[] args) throws Exception
    {
        //10000循环，没有触发C2 compilation
        for (int i = 0; i < 15000; i++)
        {
            assign(i);
        }
        Thread.sleep(1000);
    }
}
```
```
0x00007fe18910b297: mov    DWORD PTR [r10+0x68],r11d
                                                ;*putstatic field1
                                                ; - TestJIT::assign@3 (line 13)

  0x00007fe18910b29b: mov    DWORD PTR [r10+0x7c],r9d  ;*putstatic field6
                                                ; - TestJIT::assign@34 (line 18)

  0x00007fe18910b29f: mov    DWORD PTR [r10+0x78],ecx  ;*putstatic field5
                                                ; - TestJIT::assign@27 (line 17)

  0x00007fe18910b2a3: mov    DWORD PTR [r10+0x74],ebx  ;*putstatic field4
                                                ; - TestJIT::assign@21 (line 16)

  0x00007fe18910b2a7: mov    DWORD PTR [r10+0x70],esi  ;*putstatic field3
                                                ; - TestJIT::assign@15 (line 15)

  0x00007fe18910b2ab: mov    DWORD PTR [r10+0x6c],r8d  ;*putstatic field2
                                                ; - TestJIT::assign@9 (line 14)
```
可以看到compiler把field[1-6]的赋值，重排序成field[1,6,5,4,3,2].

当把filed1和field6加上volatile后，生成的汇编如下：
```
0x00007f29d910b197: mov    DWORD PTR [r10+0x68],r11d
                                                ;*putstatic field1
                                                ; - TestJIT::assign@3 (line 13)

  0x00007f29d910b19b: mov    DWORD PTR [r10+0x6c],esi  ;*putstatic field2
                                                ; - TestJIT::assign@9 (line 14)

  0x00007f29d910b19f: mov    DWORD PTR [r10+0x78],r9d  ;*putstatic field5
                                                ; - TestJIT::assign@27 (line 17)

  0x00007f29d910b1a3: mov    DWORD PTR [r10+0x74],ecx  ;*putstatic field4
                                                ; - TestJIT::assign@21 (line 16)

  0x00007f29d910b1a7: mov    DWORD PTR [r10+0x70],ebx
  0x00007f29d910b1ab: mov    DWORD PTR [r10+0x7c],r8d
  0x00007f29d910b1af: lock add DWORD PTR [rsp],0x0  ;*putstatic field6
                                                ; - TestJIT::assign@34 (line 18)
```
可以看到中间没有加volatile的field[2,3,4,5]，已经被compiler重排序, 而field1和field6还是在头和尾，这里还有一点比较有意思field1的lock add内存屏障已经被优化掉，只要field1和field6不重排，field1的volatile可以省略掉．

当把filed1和field6改成AtomicInteger, 使用lazySet进行赋值时，其汇编如下，
```
0x00007fa1dd112060: mov    DWORD PTR [rsp-0x14000],eax
  0x00007fa1dd112067: push   rbp
  0x00007fa1dd112068: sub    rsp,0x10           ;*synchronization entry
                                                ; - TestJIT::assign@-1 (line 13)

  0x00007fa1dd11206c: mov    r10d,esi
  0x00007fa1dd11206f: shl    r10d,1             ;*ishl
                                                ; - TestJIT::assign@5 (line 13)

  0x00007fa1dd112072: mov    r11,0x78c7dab20    ;   {oop(a 'java/lang/Class' = 'TestJIT')}
  0x00007fa1dd11207c: mov    r8d,DWORD PTR [r11+0x68]  ;*getstatic field1
                                                ; - TestJIT::assign@0 (line 13)

  0x00007fa1dd112080: test   r8d,r8d
  0x00007fa1dd112083: je     0x00007fa1dd1120d1
  0x00007fa1dd112085: mov    DWORD PTR [r12+r8*8+0xc],r10d
                                                ;*invokevirtual putOrderedInt
                                                ; - java.util.concurrent.atomic.AtomicInteger::lazySet@8 (line 110)
                                                ; - TestJIT::assign@6 (line 13)

  0x00007fa1dd11208a: mov    r10d,DWORD PTR [r11+0x6c]
                                                ;*getstatic field6
                                                ; - TestJIT::assign@33 (line 18)

  0x00007fa1dd11208e: mov    ebp,esi
  0x00007fa1dd112090: shl    ebp,0x6            ;*ishl
                                                ; - TestJIT::assign@39 (line 18)

  0x00007fa1dd112093: mov    r9d,esi
  0x00007fa1dd112096: shl    r9d,0x2
  0x00007fa1dd11209a: mov    DWORD PTR [r11+0x70],r9d  ;*putstatic field2
                                                ; - TestJIT::assign@12 (line 14)

  0x00007fa1dd11209e: mov    r9d,esi
  0x00007fa1dd1120a1: shl    r9d,0x5
  0x00007fa1dd1120a5: mov    DWORD PTR [r11+0x7c],r9d  ;*putstatic field5
                                                ; - TestJIT::assign@30 (line 17)

  0x00007fa1dd1120a9: mov    r9d,esi
  0x00007fa1dd1120ac: shl    r9d,0x4
  0x00007fa1dd1120b0: mov    DWORD PTR [r11+0x78],r9d  ;*putstatic field4
                                                ; - TestJIT::assign@24 (line 16)

  0x00007fa1dd1120b4: shl    esi,0x3
  0x00007fa1dd1120b7: mov    DWORD PTR [r11+0x74],esi  ;*putstatic field3
                                                ; - TestJIT::assign@18 (line 15)

  0x00007fa1dd1120bb: test   r10d,r10d
  0x00007fa1dd1120be: je     0x00007fa1dd1120e5
  0x00007fa1dd1120c0: mov    DWORD PTR [r12+r10*8+0xc],ebp
                                                ;*invokevirtual putOrderedInt
                                                ; - java.util.concurrent.atomic.AtomicInteger::lazySet@8 (line 110)
                                                ; - TestJIT::assign@40 (line 18)
```
没有添加任何内存屏障(lock add, mfence), 只保证field1和field6的顺序，不会重排．

[1] http://stackoverflow.com/questions/7805192/is-a-volatile-int-in-java-thread-safe
[2] http://codingcat.me/2015/05/09/big-brain-hole/
[3] http://www.infoq.com/cn/articles/ftf-java-volatile
[4] http://www.infoq.com/cn/articles/zzm-java-hsdis-jvm
[5] Intel® 64 and IA-32 Architectures Software Developer’s Manual
[6] http://jpbempel.blogspot.com/2013/05/volatile-and-memory-barriers.html
