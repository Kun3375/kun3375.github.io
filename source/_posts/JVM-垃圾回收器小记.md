---
title: JVM 垃圾回收器小记
author: 斯特拉不用电
date: 2019-02-04 00:57:43
tags:
  - Java
  - JVM
categories: Java
comments: true
---
### 能力各异的垃圾回收器 ###

#### Serial #####

Serial 收集器是一个新生代串行垃圾回收器，单线程执行，使用复制算法，整个流程会 STW。在单核环境下可能会不存在线程切换的问题而有略高的效率。也是之前 client 模式下的默认垃圾回收器。

#### ParNew ####

ParNew 收集器是一个新生代并行垃圾回收器，多线程执行，使用复制算法，整个流程会 STW。除了使用多个线程进行垃圾回收之外，其余和 Serial 一致。
<!-- more -->
#### Parallel Scavenge ####

Parallel Scavenge 收集器是一个新生代并行垃圾回收器，多线程执行，使用复制算法，整个流程会 STW。其设计的初衷是为了提高系统吞吐量（应用程序时间占比）。相对后文中的 CMS 尽管缩短了用户线程的停顿时间但是两次扫描会拉长整个垃圾回收的时间。

#### Serial Old ####

Serial Old 是一个老年代串行垃圾回收器，使用单线程的标记整理算法。类似 Serial 的老年代版本，常用于 client 模式。

#### Parallel Old ####

Parallel Old 是一个老年代并行垃圾回收器，使用多线程执行标记整理算法。

#### Concurrent Mark Sweep ####

Concurrent Mark Sweep 垃圾收集器，简称 CMS，一个近乎并发执行的垃圾回收器，其设计的初衷就是尽可能地缩短用户线程的停顿时间。它分为 4 个阶段：
1. 初始标记（Initial Mark）标记所有 GC Roots 关联的对象，速度十分快。
2. 并发标记（Concurrent Mark）沿 GC Roots 搜索对象，判断是否存活。
3. 重新标记（Remark）标记并发阶段出现的对象。
4. 并发清除（Concurrent Sweep）对垃圾对象清理。

在初始标记和重新标记阶段仍然需要暂停用户线程（STW），但是速度很短，所以一般视 CMS 为一个并发回收器。尽管 CMS 停顿时间少，但是它依然有着显著的缺点：
1. 吞吐量降低。并发标记和清理会会和用户线程抢占 CPU 资源，而且这两个阶段持续的时间会相对较长。
2. 无法处理浮动垃圾。由于 CMS 线程和用户线程并行工作，所以期间产生的一些垃圾会无法回收，直到下一次 GC。同时 CMS 模式下需要在老年代预留一定的空间，而不能等到近乎填满之后才启动。如果 Concurrent Mode Failure，那么会触发一次 SerialOld FullGC。
3. 产生内存碎片。这是由于 CMS 使用标记清除算法的原因。最后严重的堆碎片化可能因为无法容纳一个大对象而被迫提前进行一个 FullGC。对此，可以控制 JVM 在一定次数的 CMS GC 之后进行一次碎片整理。

#### Garbage First ####

Garbage First 简称 G1，是 JDK7 后引入的一个分代收集器，在 JDK9 已是默认选项。
感觉有必要关于 G1 新开笔记，待更。

### 相关参数和注意 ###
| 参数 | 效果 |
|:-----:|:-----:|
|**通用参数**||
|`-XX:+PrintCommandLineFlags`|打印当前 JVM 定制的参数，可查看使用的垃圾回收器|
|`-XX:+PrintGC` `-XX:+PrintGCDetails`|打印 GC 日志，精简 or 详细|
|`-XX:+PrintGCApplicationStoppedTime`|打印 GC 时暂停的时间|
|`-XX:+PrintGCApplicationConcurrentTime`|打印 GC 间应用运行的时间|
|`-XX:+PrintGCTimeStamps`|打印 GC 阶段触发的时间戳|
|`-XX:+PrintHeapAtGC`|在 GC 时打印堆详情|
|`-XX:+DisableExplicitGC`|禁用程序中显式的提示 FullGC 触发：System.gc()，有 OOM 风险，许多框架使用永久区内存，只能通过 FullGC 调用 sun.misc.Cleaner 进行回收|
|**回收器选择**||
|`-XX:+UseSerialGC`|使用串行垃圾回收器，Serial+SerialOld(MSC)|
|`-XX:+UseParallelGC`|Young 使用并行回收器(Parallel Scavenge)，Old 默认串行回收，可以搭配 ParOld 而无法搭配 CMS|
|`-XX:+UseParNewGC`|Young 使用并行回收器(Parrallel New), 可以搭配 CMS 使用|
|`-XX:+UseParallelOldGC`|Old 使用并行回收器（Parallel Old），一般搭配 ParScvg 提高系统吞吐量|
|`-XX:+UseConcMarkSweepGC`|Old 使用并行回收器（CMS），默认 Young 使用 ParNew，备用 SerialOld 进行 FullGC|
|`-XX:+UseG1GC`|使用 G1 回收器|
|**ParallelYoungGC 配置**||
|`-XX:ParallelGCThreads`|配置 Young 区的并行收集线程数，默认 `(ncpus <= 8) ? ncpus : 3 + ((ncpus * 5) / 8)`|
|**Parallel Scavenge 涉及配置**||
|`-XX:GCTimeLimit`|设置 GC 时间上限，默认 98，超出后抛出 OOM|
|`-XX:GCHeapFreeLimit`|设置非 GC 中的时间阈值，默认 2，低于该值会抛出 OOM|
|`-XX:MaxGCPauseMillis`|设置年轻代回收停顿的最大毫秒数，如果在 GC 时超过该阈值，JVM 会尝试调整堆空间的配比，处理优先级高|
|`-XX:GCTimeRatio`|配置 GC 时间的比重 1/(1 + N)，如果过长 JVM 会调整堆空间的配比，处理优先级低于 MaxGCPauseMillis，但是高于其他空间配置|
|`-XX:+UseAdaptiveSizePolicy`|开启堆空间自适应调整，推荐在 Parallel Scavenge 下启用，如果需要手动调整堆空间配比，请使用 `-` 停用|
|**CMS 相关配置**||
|`-XX:+CMSParallelInitialMarkEnabled`|使 CMS 的初始化标记阶段并行进行，1.8 已并行处理，该选项针对 1.5~1.7 |
|`-XX:ParallelCMSThreads`|设置 CMS 回收线程数量，默认为 (Young 并行回收线程 ParallelGCThreads + 3)/4|
|`-XX:CMSWaitDuration`|设置 CMS 扫描线程的间隔时间，默认 2000 ms|
|`-XX:CMSInitiatingOccupancyFraction`|设置 CMS **首次**触发回收的堆占用百分比，1.7 之后默认 92，后续的回收触发比例由 JVM 自行控制|
|`-XX:+UseCMSInitiatingOccupancyOnly`|设置每次 CMS 触发的堆占用比例都沿用 CMSInitiatingOccupancyFraction 设定值|
|`-XX:+CMSScavengeBeforeRemark`|设置 CMS 启动前进行一次 YoungGC，以减轻重新标记阶段时候的工作量，减少暂停时间，需要斟酌|
|`-XX:+UseCMSCompactAtFullCollection`|允许在内存不够时进行碎片整理，碎片整理时无法并发，仅 CMS 启用时有效，默认开启|
|`-XX:+CMSFullGCsBeforeCompaction`|多少次 FullGC 之后进行碎片压缩，默认 0，每次进行压缩|
|`-XX:+CMSClassUnloadingEnabled`|针对 1.6~1.7，允许 CMS 清理永久区，对不再使用的类进行清理，需要斟酌，代替更早之前的 CMSPermGenSweepingEnabled|
|`-XX:+CMSInitatingPermOccupancyFraction`|针对 1.7 及之前，针对永久区控制触发 CMS 的阈值，效果同 CMSInitiatingOccupancyFraction |
|`-XX:+ExplicitGCInvokesConcurrent` `-XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses`|使用 CMS 来执行 FullGC，DisableExplicitGC 的 OOM 风险可以使使用该命令来避免|
|**G1 相关配置**||
|`-XX:+UseStringDeduplication`|优化字符串空间，去除冗余字符串|
|`-XX:StringDeduplicationAgeThreshold`|字符串去重会针对年龄大的字符串对象，而该值则控制这个年龄阈值，默认 3|
|`-XX:+PrintStringDeduplicationStatistics`|打印 StringDeduplication 的触发情况|
|**关于 GC 日志记录**
|`-Xloggc:<FilePath>`|记录 GC 日志，设定日志目录。如果需要虚拟机全部日志信息需要使用 `XX:+LogVMOutput` 以及 `-XX:LogFile=<FilePath>`|
|`-XX:+UseGCLogFileRotation`|开启 GC 日志的滚动|
|`-XX:NumberOfGCLogFiles=<N>`|设置 GC 滚动日志的文件个数，N >= 1|
|`-XX:GCLogFileSize=N`|设置 GC 每个滚动日志的文件大小，N >= 8KB|
