---
title: G1GC 解读
date: 2019-06-24 18:42:37
tags: 
  - JVM
  - Java
categories: Java
author: 斯特拉不用电
comments: true
---

之前对 G1 之前的垃圾回收器进行了整理[《JVM 垃圾回收器小记》](http://kun3375.com/2019/02/JVM-垃圾回收器小记/)，但是留下了 G1 的坑长久未填，现在把相关的概念、配置参数以及 G1 的流程进行了整理。

## 基本概念 ##

### Region ###
不同于之前的垃圾回收按照连续的物理内存空间进行划分产生 Yong、Tenured、MetaSpace 区域并使用分代垃圾回收处理。G1 将整个堆空间切分成若干个小型的区域 Region 来存放对象，每个 Region 可以单独作为 *Yong*、*Tenured* 或者 *Humongous*（大对象分配区域），这使得不同代的内存区域在物理上是可以割裂的。

<!-- more -->

    Humongous 区域特性：
    - 一块 Humongous 区域不止占用一个 Region 基本大小，因为那些大于等于 Region 基本大小一半的对象都会被分配到 Humongous 区域。
    - Humongous 大对象直接作为 Tenured，在 GCM Cleanup 阶段或者 FullGC 时候被回收。
    - 大对象分配前会检查 InitiatingHeapOccupancyPercent 数值和 MarkingThreshold，超过则启动 GCM，避免 FullGC 和分配失败的可能。

### Card Table ###
卡表，这个概念也适用于 CMS GC 并发垃圾回收。堆区空间会被划分成一个个 512 Byte 的卡，并维护一个卡表，保存一个标识位来标识对应卡区是否可能持有有年轻代的引用。目的是减少在 YongGC 时候对于整个 Tenured 区域的扫描。如果可能存在 Yong 区的引用则称为 **Dirty Card**（**脏卡**）。

### Remembered Set ###
简称 RSet，基于 Card Table 概念实现。对于 Region 而言，Card Table 记录了是否引用 Young 区对象，而 RSet 则是记录了其他 Card 引用本 Region 的信息。RSet 是一个哈希表，键为引用本 Region 的其他 Region 的起始地址，值为引用了对应 Region 中对应 Card 的集合。
- 效益：G1 回收部分 Region，YongGC 时候可以通过 RSet 找到跨代引用的 Tenured Region，MixedGC 时候可以通过全部 YongRegion 和 Tenured RSet 得到，不用全堆扫描。
- 维护：write barrier：写入屏障，即时编译生成的机器码中对于所有引用的更新都会生成额外的逻辑，来记录 Card Table 和 RSet 的改变。

### Collection Set ### 
简称 CSet，所有需要被回收的对象。

### Pause Prediction Model ###
停顿预测模型。G1 是响应时间优先的算法，和 CMS 不同，它可以设置用于期望停顿时间，通过 MaxGCPauseMillis（这是一个软要求 ）设定。G1 根据历史回收数据和 MaxGCPauseMillis 来判断需要回收的 Region 数量。其以衰减标准偏差为理论基础实现，具体数学内容不展开。

### Float Garbage ###
由于 G1 基于启动时的存活对象的快照（Snapshot At The Begining，SATB）所有在收集过程中产生的对象会被视为存活对象，无法识别垃圾。这部分对象只会在下一次 GC 时候被清理，被称为 Float Garbage。

## 标记方案 ##

**Global Concurrent Marking**：并发标记本身为 MixedGC 提供对象标记服务，但是它的发生是随着 YoungGC 而开始的，总共几个阶段：
- 初始标记（Initial Mark，STW）。它标记了从 GC Root 开始直接可达的对象（Root Trace），因为需要暂停所有应用线程（STW）代价较大，它会复用 YoungGC 的 Root Trace。
- 根区域扫描（Root Region Scan）标记了从 GC Roots 开始可达的老年代对象。
- 并发标记（Concurrent Marking）。这个阶段从 GC Root 开始对堆中的对象标记，标记线程与应用程序线程并行执行，并且收集各个 Region 的存活对象信息，这个步骤可以被新的 YoungGC 打断。
- 最终标记（Remark，STW）。标记那些在并发标记阶段发生变化的对象，将被回收。 
- 清除垃圾（Cleanup）。执行最后的清理工作，清除空 Region（没有存活对象的），并把存活对象进行移动，减少 Region 的碎片化。

## 回收方案 ##

- **YoungGC**
也就是 MinorGC，指定所有的年轻代 Region 进行回收。通过控制年轻代 Region 个数（内存大小）来控制 YoungGC 的消耗时间。
- **MixedGC**
其实 MixedGC 是作为 YoungGC 的升级，和 YoungGC 的区别在于使用了不同的 CSet，在 MixedGC 过程会包含若干 Tenured Region。在老年代对象占用堆区的内存达到阈值 `InitiatingHeapOccupancyPercent` 时会触发标记，并在标记结束时切换成 MixedGC。纳入每次 MixedGC 中除了 Yong Region，Tenured Region 数量会由期望停顿时间 `MaxGCPauseMillis` 估算出来。剩余的 Tenured Region 的部分会在下一次 MixedGC 中被回收。一次标记后 MixedGC 的最大次数由 `G1MixedGCCountTarget` 控制。
- **SerialOldGC**
根据 MixedGC 的执行行为可以了解到，如果垃圾产生的速度超过 MixedGC 速度，JVM 有必要采取额外的措施进行垃圾回收了，会使用一次 SerialGC 进行完全回收（FullGC）。还会触发 FullGC 的 MetaSpace 的使用。（关于 MetaSpace 推荐文章 [《MetaSpace解读》](http://lovestblog.cn/blog/2016/10/29/metaspace/，精简细致，不再搬运) 了解

## 重要参数 ##
- -XX:G1HeapWastePercent：允许浪费的堆空间阈值。在 Global Concurrent Marking 结束之后，我们可以知道老年代 Region 中有多少空间要被回收，在每次 YGC 之后和再次发生 MixedGC 之前，会检查垃圾占比是否达到此参数，只有达到了，下次才会发生 MixedGC。
- -XX:G1MixedGCLiveThresholdPercent：老年代 Region 中的存活对象的占比，只有在此参数之下，才会被选入 CSet，默认 65%。
- -XX:G1MixedGCCountTarget：一次 Global Concurrent Marking 之后，最多执行 MixedGC 的次数。
- -XX:G1OldCSetRegionThresholdPercent：一次 MixedGC 中能被选入 CSet 的最多老年代 Region 数量，默认堆的 10%。
- -XX:G1HeapRegionSize：设置Region大小，并非最终值，默认会自动计算出一个合适值。
- -XX:MaxGCPauseMillis：设置G1收集过程目标时间，默认值200ms，软限制。
- -XX:G1NewSizePercent：新生代最小值，默认值5%。
- -XX:G1MaxNewSizePercent：新生代最大值，默认值60%。
- -XX:ParallelGCThreads：STW期间，并行GC线程数。可以不进行指定，默认会使用 CPU 支持的线程数（如果线程数小于等于 8），或者按 8 + 线程数 * 调整值 5/8 或 5/16（线程数大于 8）。
- -XX:ConcGCThreads：并发标记阶段，并行执行的线程数，为 ParallelGCThreads/4。
- -XX:InitiatingHeapOccupancyPercent：设置触发标记周期的 Java 非年轻代堆占用率阈值。默认值是 45%。
- -XX:G1ReservePercent=10：空闲空间预留内存百分比。

参考
- https://liuzhengyang.github.io/2017/06/07/garbage-first-collector/
- https://tech.meituan.com/2016/09/23/g1.html
- https://www.oracle.com/technetwork/cn/articles/java/g1gc-1984535-zhs.html
