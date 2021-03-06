---
title: 从另一个角度了解JAVA的可见性
date: 2019-06-07 23:30:46
tags:
---
# 什么是可见性问题
在多线程环境下，一个线程对某个共享变量更新之后，其它线程访问该变量的线程，是否可以立刻读取到这个变量的更新结果，或者说，线程A对共享变量的修改，是否对线程B可见。这就是线程安全问题的另一个表现形式，可见性。
# 为什么出现这样的问题       
 线程是运行在处理器上，现在的计算机大多数都是多核的，计算机有主内存（RAM）、每个处理器有自己的写缓冲器（Store Buffer)、高速缓存，由于读取速度不匹配，所以处理器并不是直接与主内存打交道，内存的读写操作都是通过寄存器、高速缓存、写缓冲器、无效化队列等部件执行内存读写的。这些部件相当于主内存数据的副本，副本存储的份数越多，数据一致性越难保证。多个处理器读写数据也是优先读写当前处理器的副本数据，这就可能导致多个副本之间数据不一致。
> 既然有这样的问题，处理器是如何屏蔽的呢，这就引入了缓存一致性协议，也就是俗称的缓存同步。缓存同步使得一个处理器上可以读取到另一个处理器对共享变量所做的更新。在这里缓存一致性读者可以下来看一下MESI协议。

所以，缓存同步的是可以保障可见性的。

>对变量的更新缓存的层级关系如下：
>       __写缓冲器  ->  高速缓存  ->  主内存__

下面我们看两个定义
> * 处理器对**共享变量**所做的更新操作，从写缓冲器中写入处理器的高速缓存或者主内存中，这个动作被称为**冲刷处理器缓存**
> * 处理器读取**共享变量**时，如果其它处理器更新了该值，那么处理器要从其它处理器的高速缓存或者主内存中进行同步相应的变量值到本缓存，这个动作被称为**刷处理器缓存**
- 对于上面两个定义，我们提取一下关键字：
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 
        **共享变量**     **更新（写）**      **读取（读）** 
       
 看到这里，我们其实主要提到的关键字是共享变量，主要的操作是读写，也就是说对于共享变量的读写才会有可见性问题。
> _处理器会在写共享变量的时候冲刷处理器，读取共享变量的时候刷新处理器缓存，**保证对变量的更新已经同步到高速缓存或者主内存，然后通过MESI协议，保证共享变量在多线程的可见性。**_

 从这个角度来分析，只要我们告诉处理器，谁是共享变量，处理器就可以保证该变量的可见性了
>　但是遗憾的是，JAVA中的变量都是JVM堆中存储的，无法告诉处理器哪个变量是共享变量。但是既然已经知道了处理器是怎么保证可见性的， 那么我们是不是可以定义一个关键字，对于这个关键字做下面这样的处理，就可以保证可见性呢

    读的时候去刷处理器缓存，写的时候去冲刷处理器缓存

java中就是这样处理的，在java中，有一个关键字__volatile__大家一定都知道，怎么去理解它
>* 被volatile标识的变量，其实就是告诉处理器，该变量是共享变量，需要保证该变量的可见性
>*  另一方面，被volatile修饰的变量，可以告诉JIT编译器该变量可能会被共享，优化时悠着点儿。
    
从CPU指令角度来看volatile关键字的实现 ，其实是使用到的**内存屏障**，通过内存屏障保证在**共享变量写的时候冲刷处理器**，**共享变量读的时候刷新处理器缓存**。volatile关键字的内存屏障具体是如何实现的， 后续会有文章更新，希望大家继续关注我。
    
> 到这里，希望看到本文章的读者是，可以从另一个角度去认识JAVA中的volatile关键字。

##结尾再废话两句
在说起可见性的时候，我们说到了多线程，说到多线程我们一定会与多个处理器想到一起，**其实并非在多个处理器下才会出现可见性问题， 在单核处理器也会出现可见性问题**
> 单个处理器，在线程切换的时候 ，当前线程对共享变量的修改会作为线程的上下文保存起来，这就可能导致其它处理器无法看到该线程对共享变量的更新（无法看到该变量的相对新值）。


    下面可能可能会写一篇关于JAVA中有序性的文章，请大家继续关注 
--- 
希望大家可以继续关注我，一起成长，记录下成长路上的点点滴滴。