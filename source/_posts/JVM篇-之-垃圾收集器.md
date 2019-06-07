---
title: JVM篇 之 垃圾收集器
date: 2019-06-07 22:44:26
tags:
---
##前言
 > 本篇讲一下各种GC之间的细微差异讲的清楚的，希望通过本篇文章，读者可以对GC的种类有更深刻的了解。
对于GC算法的可参数文章[JVM篇之 GC算法](https://www.jianshu.com/p/2ae9a0c9790a)

# 一、串行收集器-Serial
Serial收集器最古老的，最稳定的，历经考验，内部的BUG比较少的一个收集器，单个GC线程进行垃圾回收。
> 会作为CMS收集器降级处理时会使用此收集器。
#### JVM GC参数
    增加JVM参数  -XX:+UseSerialGC 使用串行收集器，启用此参数后
    - 新生代、老年代使用串行回收
    - 新生代使用复制算法
    - 老年代使用标记-压缩
#### 回收过程如下图：
![串行收集器.png](http://psqhc4tjv.bkt.clouddn.com/1.png)
通过上图，可以看出在应用程序线程到达安全点后，会全部暂停，然后运行GC线程(单线程)，GC线程线程执行完成之后，再开始执行应用线程。
> 缺点：因为是串行的，只使用一个线程进行回收，所以会产生较长的停顿时间，在多核的CPU上，是无法发挥CPU的性能
#### GC日志 
![Serial收集器GC日志 .png](http://psqhc4tjv.bkt.clouddn.com/2.png)
> 当看GC日志有红色加粗显示的标识时，说明当前使用的是Serial收集器GC
# 二、并行收集器-ParNew
Serial收集器**新生代**的**并行**版本，与Serial收集器相比只影响新生代的回收，此收集器需要多核cpu的支持，GC性能会更好。
#### JVM GC参数
    增加JVM参数  -XX:+UseParNewGC ，启用ParNew收集器，启用之后
    -新生代并行
    -新生代使用复制算法
    -老年代串行
    -老年代使用标记-压缩算法
由于新生代是并行回收，可以通过参数来调整并行回收的线程数量
####      
      -XX:ParallelGCThreads 限制线程数量

> ParNew收集器与Serial收集器主要区别在于，新生代的回收，前者是多线程回收，后者是单线程回收
#### 回收过程如下图：
![ ParNew收集器.png](http://psqhc4tjv.bkt.clouddn.com/3.png)
通过上图，可以看出在应用程序线程到达安全点后，会全部暂停，然后运行**多个GC线程**去做垃圾回收，GC线程线程执行完成之后，再开始执行应用线程。
#### GC日志 
 在GC日志中出现上图中红色加粗显示的标识时，说明当前GC使用的是ParNew收集器,如下图显示![ParNew GC日志.png](http://psqhc4tjv.bkt.clouddn.com/4.png)
 
# 三、Parallel收集器
- Serial收集器在新生代和老年代的多线程版收集器
- ParNew收集器，在老年代的并行版收集器
# 
    - 新生代复制算法
    - 老年代 标记-压缩
    - 更加关注吞吐量
#### JVM GC参数
    -XX:+UseParallelGC 
      - 新生代使用Parallel收集器 
      - 老年代串行
    -XX:+UseParallelOldGC
      - 新生代使用Parallel收集器
      - 并行老年代
 #### 回收过程如下图   
![图片.png](http://psqhc4tjv.bkt.clouddn.com/5.png)
#### GC日志 
 在GC日志中出现上图中红色加粗显示的标识时，说明当前GC使用的是Parallel收集器,如下图显示!![Parallel GC日志.png](http://psqhc4tjv.bkt.clouddn.com/6.png)
#### 其它JVM参数
由于Parallel收集器比较关注吞吐量，所以还提供了一些配套的参数来控制GC的时间和占比
# 
    -XX:MaxGCPauseMills（停顿时间）
        设置GC的最大停顿时间，单位毫秒
        GC尽力保证回收时间不超过设定值
        作为一个GC的目标值
    -XX:GCTimeRatio（可理解为吞吐量）
        GC所使用的CPU时间占用总时间的百分比
        在0-100的取值范围
        默认99，即最大允许1%时间做GC
通常情况下，我们希望GC停顿时间短 ,同时又希望吞吐量高，但事实上这两个参数是矛盾的。因为停顿时间和吞吐量不可能同时调优。在GC工作负载不变的情况下
- 如果提高GC的频率，因为频率高了，所以每次GC要处理的垃圾就少了，GC速度自然速度就会加快，但是频繁的GC会对系统的整体性能有损伤的，就会出现GC停顿时间短了，但是系统整体性能并不会很好。 
- 如果降低GC的频率，自然而然每次要处理的垃圾就会增加，所以会导致每次GC的时间相对就会增加，但是对系统的整体性能是有所上升的。
###### 对吞吐量的理解
- cpu分到应用程序上的时间越多，吞吐量自然就会增加
- cpu分到GC线程上的时间越多，处理应用线程就会越少。
所以吞吐量，可以用应用线程占用CPU时间的长短来衡量。

在停顿时间和吞吐量两者上，在GC工作负载不变的情况下，不可能两者同时提高，个人认为除非优化GC算法，从根本上降低GC的工作负载。
> 所以在调试停顿时间 和吞吐量这两个参数时，要根据实际情况来调整。
# 四、CMS收集器
CMS(Concurrent Mark Sweep) 并发标记清除，并发的意思是与用户线程一起运行，从名字上可以理解，此算法是使用标记-清除算法，比较关注停顿时间 。
#### JVM GC参数
    增加JVM参数  -XX:+UseConcMarkSweepGC，启用CMS收集器，启用之后
    - 新生代使用ParNew收集器
    - 新生代使用复制算法
    - 老年代使用CMS收集器
    - 老年代使用并发标记-清除
#### 回收过程
因为CMS收集器要与用户线程一起运行，所以它的算法和实现机制比较复杂，主要工作可以分为五步：
- 初始标记
```
- 标识根节点直可达的对象
- 速度快
- 会产生STW
```
- 并发标记（和用户线程一起）
```
-  并发标记存活对象
- 占用时间相对较长
```

- 重新标记
```
- 由于并发标记时，用户线程依然运行因以在正式清理前，对有变化的对象，所以需要修正，对对象重新标记
- 会产生STW
```
-  并发清除（和用户线程一起）
```
-  基于标记结果，直接清理不可达的对象
```
- 并发重置
```
- 为下一次CMS GC做准备
```
##### 回收过程如下
![cms回收过程.png](http://psqhc4tjv.bkt.clouddn.com/7.png)
#### GC日志 
     CMS-initial-mark  初始标记
     CMS-concurrent-mark  并发标记
     CMS-remark   重新标记
     CMS-concurrent-sweep  并发清除
     CMS-concurrent-reset  并发重置
具体的日志内容如下:![图片.png](http://psqhc4tjv.bkt.clouddn.com/8.png)

###### cms收集器的特点
- 尽可能降低停顿
- 会影响系统整体吞吐量和性能
    比如，在用户线程运行过程中，分一半CPU去做GC，系统性能在GC阶段，反应速度就下降一半
- 清理不彻底
因为在清理阶段，用户线程还在运行，会产生新的垃圾，无法清理
- 与标记-压缩相比会产生内存碎片 
- 并发阶段会降低吞吐量

因为CMS收集器和用户线程一起运行，所以不能在空间快满时再清理，因为与用户线程同时进入，如果在快满的时候进行清理的话，很容易出两用户线程申请内存，出现concurrent mode failure的情况，所以JVM考虑到这点，提供相应的参数进行设置触发GC的阈值
# 
        -XX:CMSInitiatingOccupancyFraction设置触发GC的阈值
        -XX:CMSInitiatingPermOccupancyFraction：当永久区占用率达到这一百分比时，启动CMS回收
        -XX:CMSInitiatingOccupancyFraction：设置CMS收集器在老年代空间被使用多少后触发
 即使设置了阈值也不能保证不会出现并发收集的错误 ，如果不幸内存预留空间不够，就会引起concurrent mode failure，当发生这种情况之后，CMS收集器会降级到Serial收集器进行垃圾回收，这时候会暂停用户线程，会产生一个长时间的停顿来回收垃圾。
下图是发生concurrent mode failure时的GC日志![concurrent mode failure.png](http://psqhc4tjv.bkt.clouddn.com/9.png)

## 有关碎片
因为CMS采用标记清除算法，所以会产生大量的内存碎片，所以JVM提供了两个参数来对碎片进行整理
#
        -XX:+ UseCMSCompactAtFullCollection 
            Full GC后，进行一次整理
            整理过程是独占的，会引起停顿时间变长（因为要移动大量的存活对象)
        -XX:+CMSFullGCsBeforeCompaction 
            设置进行几次Full GC后，进行一次碎片整理
        -XX:ParallelCMSThreads 
           设定CMS的线程数量
      

# 五、各GC收集器对比
| 收集器| JVM参数           | 新生代        | 老年代           |
| ------------- |:-------:| :-------------------:| :------------------:|
| Serial      |  -XX:+UseSerialGC |串行、复制算法|串行、标记压缩算法|
| ParNew      |-XX:+UseParNewGC|并行、复制算法|串行、标记压缩算法|
| Parallel |-XX:+UseParallelGC  -XX:+UseParallelOldGC   |串行或并行、复制算法|串行或者并行、标记压缩算法|
| CMS | -XX:+UseConcMarkSweepGC  | 并行、复制算法|并发、标记清除算法|

## 六、疑问
- 为什么CMS不直接使用标记-压缩算法呢?

> 作者：BK
https://www.jianshu.com/u/a5230c4f0b7a
鉴于本人才疏学浅,不足之处还望斧正,也欢迎关注我，无特殊说明的都是自己一字一句码出来的，尊重原创，如果转载请说明出处！