---
title: 对 Netty ByteBuf 的探究
author: 斯特拉不用电
date: 2020-02-24 16:57:45
tags:
    - Netty
    - Java
categories: Java
comments: true
---
## 对 Netty ByteBuf 的探究 ##

`ByteBuf` 是 Netty 在通信过程中数据传递的容器，类似于 Java NIO 中的 `ByteBuffer`，是通道 `Channel` 传输过程中的介质。

### ByteBuf 类型 ###

从池化与否上看，分为池化的 `PooledByteBuf` 和非池化的 `UnpooledByteBuf`，

而从内存属性上看 `ByteBuf` 有着使用**堆内存**的 `HeapByteBuf` 以及**基于直接内存映射**的 `DirectByteBuf`。Netty 的高效也体现在 `ByteBuf` 的使用上，以 `NIOEventLoop` 来举例，在通道读写是后使用了 `PooledDirectByteBuf` 将字节缓冲的高效发挥到了极致。

#### ByteBuf 分配方式 ####

- **`UnPooledHeapByteBuf`** 即在堆中新建对应大小的数组并初始化容量大小、读写索引等
-  **`UnPooledDirectByteBuf`** 对应的使用 `java.nio.ByteBuffer.allocateDirect(capacity)` 进行初始化。由于使用堆外内存，可能因为使用不当产生内存泄露，建议在调试时打开最高等级的泄露探测：`-Dio.netty.leakDetection.level=PARANOID`。
    > Enables paranoid resource leak detection which reports where the leaked object was accessed recently, at the cost of the highest possible overhead (for testing purposes only).
-  **`PooledByteBuf`** 其实和非池化的 `ByteBuf` 分配方式类似，不过 Netty 特别定制了一套池化管理方案来提高分配的效率，但是稍显复杂。

对于 `PooledByteBuf`，Netty 使用了对象池、Unsafe 操作、ThreadLocal 等技术来优化，以期提升 `ByteBuf` 的申请效率和垃圾回收效率。从 `PoolByteBufAllocator` 开始，主要关注一下 `ByteBuf` 的缓存机制和内存方案。

### ByteBuf 分配流程 ###

首先 Netty 在不同线程进行 `allocate()` 时候，会采用类 ThreadLocal 技术进行优化，以 `HeapBuffer` 为例看下 `PooledByteBufAllocator.newHeapBuffer(int,int)`，**先关注第一步**：
``` java
@Override
protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
    // 1.PoolThreadLocalCache.get() 获取线程本地的 PoolThreadCache
    PoolThreadCache cache = threadCache.get();
    PoolArena<byte[]> heapArena = cache.heapArena;

    final ByteBuf buf;
    if (heapArena != null) {
        // 2.PoolArena.allocate() 来分配内存
        buf = heapArena.allocate(cache, initialCapacity, maxCapacity);
    } else {
        // 如果没有 Arena 转而使用使用 UnpooledByteBuf
        buf = PlatformDependent.hasUnsafe() ?
                new UnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
                new UnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
    }
    // 包装一下，监听泄露
    return toLeakAwareBuffer(buf);
}
```

`threadCache` 是在构造器中初始化了的一个 `PoolThreadLocalCache` 内部类。该对象继承自 `FastThreadLocal<PoolThreadCache>`。这里的 `FastThreadLocal` 是 Netty 不满足于 JDK 方案而单独实现的线程本地缓存，从 API 角度来看使用方式基本一致。
`PoolThreadLocalCache.get()` 首先进行缓存获取，未命中的情况下也会通过 `initialValue()` 初始化一个 `PoolThreadCache`。

以 `HeapByteBuf` 为例，看下 `PoolThreadCache` 几个重要的变量（省略 direct 相关）。保存了两个相关联的 `PoolArena` 和几个 `MemoryRegionCache`，是最核心实现内存分配的组件。

``` java
// 保存了该 cache 属于哪个 arena，这是 Allocator 中多个 arena 中的一个（线程均分）
final PoolArena<byte[]> heapArena;
// 不同大小的 MemoryReginCache（tiny/samll/normal）
private final MemoryRegionCache<byte[]>[] tinySubPageHeapCaches;
private final MemoryRegionCache<byte[]>[] smallSubPageHeapCaches;
private final MemoryRegionCache<byte[]>[] normalHeapCaches;
```

这里先回头看分配器的 `newHeapBuffer(int,int)`，**关注第二步**：`ByteBuf` 是由 `PoolArena` 分配，方法参数中的 `cache` 也就是持有 `arena` 的 `PoolThreadCache`。那么来看一下 `PoolArena.allocate()`：

``` java
PooledByteBuf<T> allocate(PoolThreadCache cache, int reqCapacity, int maxCapacity) {
    // 获取 ByteBuf 的对象重用
    PooledByteBuf<T> buf = newByteBuf(maxCapacity);
    // 分配内存空间
    allocate(cache, buf, reqCapacity);
    return buf;
}
```

#### ByteBuf 对象复用 ####

这里 `newBytebuf(int)` 实现取决于 heap/direct 以及 unsafe 是否可用，但是本质依然大同小异，以 `PoolHeapByteBuf` 为例，追踪到：

``` java
static PooledHeapByteBuf newInstance(int maxCapacity) {
    // 通过 Recycler 复用对象
    PooledHeapByteBuf buf = RECYCLER.get();
    // reuse 即重置各个标志位，不再展开
    buf.reuse(maxCapacity);
    return buf;
}
```

`ByteBuf` 引用的获取依托于 `RECYCLER`, 这个 `RECYCLER` 是 `ObjectPool` 对象池的实现，并将对象的创建委托给 `Recycler` 实现：

``` java
// 这里泛型 T 为 PoolByteBuf
public final T get() {
    // 不进行重用
    if (maxCapacityPerThread == 0) {
        return newObject((Handle<T>) NOOP_HANDLE);
    }
    // 获取一个本线程的栈
    Stack<T> stack = threadLocal.get();
    // pop 出一个 handle，对应 ByteBuf.deallocate() 时候 recycler 会进行 push
    DefaultHandle<T> handle = stack.pop();
    if (handle == null) {
        handle = stack.newHandle();
        // handle 为空则委托 handle 新建 ByteBuf
        handle.value = newObject(handle);
    }
    // 如果有使用过的 handle，直接复用之前的 ByteBuf 对象（解构时会 push 进栈）
    return (T) handle.value;
}
```

到此为止，`ByteBuf` 对象复用的过程已经完成。

#### 内存分配 ####

分配内存空间的步骤在 `allocate(cache, buf, reqCapacity);`，由于篇幅较长省略了相关代码，简单来说，
1. 首先对需要申请的空间进行标准化。
    - capacity <= 512B，计算一个大于等于申请容量的 16B 倍数的大小
    - 512 < capacity <= pageSzie，计算一个大于等于申请容量的 512B 倍数的大小
    - capacity > pageSize，计算一个相应的 pageSize 倍数的大小
2. 再看 *normCapacity* 大小来执行不同 allocate 逻辑：
    - ***tiny***：小于 512 Bytes
    - ***small***：大于等于 512 Bytes，而小于 *pageSize*
    - ***normal***：大于等于 *pageSize* 而小于等于 *chunkSize*
    - ***huge***：超过 *chunkSize*
    
##### 缓存命中 #####
    
说一下之所以需要标准化并区分申请大小，是因为 `PoolThreadCache` 中的 `MemoryRegionCahce[]` 是按着需要划分的内存大小依次排列的，以 `tinySubPageHeapCaches` 为例，默认大小为 512/16=32，[0] 是 16B 大小的 `ByteBuf` 的引用队列，[1] 是 32B 大小的 ByteBuf 的引用队列... 依次到 496B，总共 32 个。

<p align="center">
	<img src="https://blog-1258216698.cos.ap-hongkong.myqcloud.com/Netty/MemoryRegionCaches.png" alt="MemoryMap"  width="600" />
    <em>MemoryRegionCaches</em>
</p>

small、normal 也是类似划分，只不过每个 Region 负责缓存的 `ByteBuf` 大小不同。在 `PoolArena` 中有对应的 `tinyIdx(int)` / `smallIdx(int)` / `normalIdx(int)` 来确定相应大小的 `ByteBuf` 需要的 Region 位置。

以分配一个 *tinyCapacity* 为例，`PoolArena` 需要委托 `PoolThreadCache` 执行 `cache.allocateTiny(this, buf, reqCapacity, normCapacity)` 判断缓存命中：

``` java
boolean allocateTiny(PoolArena<?> area, PooledByteBuf<?> buf, int reqCapacity, int normCapacity) {
    return allocate(cacheForTiny(area, normCapacity), buf, reqCapacity);
}

// 先用这个 cacheForTiny 用来定位 MemoryRegionCache
private MemoryRegionCache<?> cacheForTiny(PoolArena<?> area, int normCapacity) {
    // int idx = PoolArena.tinyIdx(normCapacity);
    // 这里手动内联下 PoolArena.tinyIdx(int)
    // 大小除以 16，因为 tinyRegion 每个都差了 16B
    int idx = normCapacity >>> 4;
    if (area.isDirect()) {
        return cache(tinySubPageDirectCaches, idx);
    }
    return cache(tinySubPageHeapCaches, idx);
}

public final boolean allocate(PooledByteBuf<T> buf, int reqCapacity) {
    // Entry 队列，queue 的大小即不同类型的 cacheSize，默认分别为 512,256,64
    Entry<T> entry = queue.poll();
    if (entry == null) {
        return false;
    }
    // 初始化 ByteBuf
    initBuf(entry.chunk, entry.nioBuffer, entry.handle, buf, reqCapacity);
    // 不再引用时交给 Recycler 减少 GC
    entry.recycle();
    ++ allocations;
    return true;
}
```

##### 缓存命中失败 #####

在 `PoolArena.allocate(cache, buf, reqCapacity)` 方法中，如果缓存命中，直接返回;如果未命中，`PoolArena` 会尝试开辟新的 `ByteBuf`。先看一下 `PoolArena` 中的一些重要成员：

``` java
private final PoolSubpage<T>[] tinySubpagePools;
private final PoolSubpage<T>[] smallSubpagePools;

private final PoolChunkList<T> q050;
private final PoolChunkList<T> q025;
private final PoolChunkList<T> q000;
private final PoolChunkList<T> qInit;
private final PoolChunkList<T> q075;
private final PoolChunkList<T> q100;
```

###### PoolSubpage 分配 ######

对于小于 *pageSize* 的 tiny/small 类型，交给 `PoolSubpage` 执行分配。

###### PoolChunkList 分配 ######

大于 *pageSize* 的空间分配交给 `PoolChunkList` 完成。`PoolChunkList` 本身是一个双向列表，`PoolArena` 中的 6 个 `PoolChunkList` 分别存储不同剩余空间的 `PoolChunk` 并依次连接。不同 `ChunnkList` 的初始化：

``` java
    q100 = new PoolChunkList<T>(this, null, 100, Integer.MAX_VALUE, chunkSize);
    q075 = new PoolChunkList<T>(this, q100, 75, 100, chunkSize);
    q050 = new PoolChunkList<T>(this, q075, 50, 100, chunkSize);
    q025 = new PoolChunkList<T>(this, q050, 25, 75, chunkSize);
    q000 = new PoolChunkList<T>(this, q025, 1, 50, chunkSize);
    qInit = new PoolChunkList<T>(this, q000, Integer.MIN_VALUE, 25, chunkSize);

    q100.prevList(q075);
    q075.prevList(q050);
    q050.prevList(q025);
    q025.prevList(q000);
    q000.prevList(null);
    qInit.prevList(qInit);
```

而 `PoolChunkList` 内部的 `PoolChunk` 同时也是一个双向列表，正好形成了这个样子：

<p align="center">
	<img src="https://blog-1258216698.cos.ap-hongkong.myqcloud.com/Netty/ChunkList.png" alt="MemoryMap"  width="500" />
    <em>PoolChunkList</em>
</p>

看一下超过 *pageSize* 时候的分配方法：
``` java
private void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
    // 在不同 chunkList 中尝试分配
    // 当然地，分配成功后会检查占用率，超过上限会被转移到下个 chunkList
    if (q050.allocate(buf, reqCapacity, normCapacity) || q025.allocate(buf, reqCapacity, normCapacity) ||
        q000.allocate(buf, reqCapacity, normCapacity) || qInit.allocate(buf, reqCapacity, normCapacity) ||
        q075.allocate(buf, reqCapacity, normCapacity)) {
        return;
    }
    
    // 最初的时候偶总是要新开 chunk
    PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);
    // PoolChunk 进行 allocate
    boolean success = c.allocate(buf, reqCapacity, normCapacity);
    assert success;
    // 最初被添加到 qInit 这个 chunkList 中
    qInit.add(c);
}
```

超过 *pageSize* 时候需要 `PoolChunk` 来进行分配，`PoolChunk` 也是内存这块儿重要的一个部分

---

#### PoolChunk ####

简单看一下 `PoolChunk` 的构造：

``` java
// 省略了部分
// 在这里对 memoryMap 进行初始化
// memoryMap/depthMap 大小为 1 << maxOrder 默认 2048
memoryMap = new byte[maxSubpageAllocs << 1];
depthMap = new byte[memoryMap.length];
// memoryMap 有效索引从 1 开始
int memoryMapIndex = 1;
// d 标识 memoryMap 树状结构的深度
for (int d = 0; d <= maxOrder; ++ d) {
    // 这里 depth 标识树的每层有多少个节点
    // d:depth => 0:1,1:2,2:4,3:8,4:16...
    int depth = 1 << d;
    // 每个节点赋值为 d 深度
    for (int p = 0; p < depth; ++ p) {
        // in each level traverse left to right and set value to the depth of subtree
        memoryMap[memoryMapIndex] = (byte) d;
        depthMap[memoryMapIndex] = (byte) d;
        memoryMapIndex ++;
    }
}
// 初始化 subPages 的大小
subpages = newSubpageArray(maxSubpageAllocs);
cachedNioBuffers = new ArrayDeque<ByteBuffer>(8);
```

##### 如果超过 *pageSize* 的分配 #####

``` java
private long allocateRun(int normCapacity) {
    // normCapacity 为大于申请容量的最小 2 次幂（类似 HashMap.capacity）
    // 默认 pageSize = 8192 则 pageShifts = 13
    // 默认 maxOrder = 11
    // 若 normCapacity = 8192，d = 11，normCapacity = 16384，d = 10
    // d 为二叉树的深度，申请的越少，就寻找越深的层数，当申请一个 pageSize 时定位到树底层
    int d = maxOrder - (log2(normCapacity) - pageShifts);
    // 转下一步
    int id = allocateNode(d);
    if (id < 0) {
        return id;
    }
    // runLength 获得改节点占用的内存大小，用来更新空闲内存大小
    freeBytes -= runLength(id);
    return id;
}
```

通过 *capacity* 得到 *d(depth)* ，然后就可以通过 *d* 深度来找到 memoryMap 中的 index 了。
``` java
private int allocateNode(int d) {
    int id = 1;
    int initial = - (1 << d); // has last d bits = 0 and rest all = 1
    // value(id) 即 memoryMap[id]
    byte val = value(id);
    // 根节点作为边界值单独判定了一次
    // memoryMap 有效元素从 1 开始，长度为 1 << maxOrder << 1 默认 4096
    // 根节点判断失败，意味着空间不够
    if (val > d) { // unusable
        return -1;
    }
    // val < d 意味着该层（包含子层）空间充裕，尝试在继续往下一层
    while (val < d || (id & initial) == 0) { // id & initial == 1 << d for all ids at depth d, for < d it is 0
        id <<= 1;
        val = value(id);
        // 判断同层余量
        if (val > d) {
            id ^= 1;
            val = value(id);
        }
    }
    byte value = value(id);
    assert value == d && (id & initial) == 1 << d : String.format("val = %d, id & initial = %d, d = %d",
            value, id & initial, d);
    // 设置 memoryMap 该位置不可用
    // unusable 在构造器中被初始化为 maxOrder + 1，默认即为 12
    setValue(id, unusable); // mark as unusable
    // 更新树上祖先节点的值，用来之后判断子节点的可用情况
    updateParentsAlloc(id);
    // 即 index of memoryMap
    return id;
}
```

用图形来展示的话 `memoryMap` 就如下所示。每个方格是其中一个元素，`allocateNode(int d)` 就是通过申请的大小确定了深度 d 之后，来寻找可用的一个内存位置，反映在 `memoryMap` 的索引号上。虽然是看上去是树形的层级关系，但是下层本质是上层的一部分。

<p align="center">
	<img src="https://blog-1258216698.cos.ap-hongkong.myqcloud.com/Netty/MemoryMap.png" alt="MemoryMap"  width="500" />
    <em>MemoryMap</em>
</p>

##### 小于 *pageSize* 的分配 #####

看需要看下 `PoolChunk` 中 `PoolSubPage[]` 的结构，每个 `PoolChunk` 都是 N 个 page，memoryMap 的叶子节点所维护的内存就是各个 page 的内存。Netty 对于小于 *pageSize* 的内存申请，首先会定位通过 `allocateNode(int)` 找到对应的一个页的内存，然后该页就被切分成若干块作为 `SubPage` 单元来使用。 

``` java
private long allocateSubpage(int normCapacity) {
    PoolSubpage<T> head = arena.findSubpagePoolHead(normCapacity);
    // memoryMap 的最底层节点都是 pageSize，所以 subPage 的分配都在这一层
    int d = maxOrder;
    synchronized (head) {
        // 这里上面已经分析了，获取 memoryMap 索引号
        int id = allocateNode(d);
        if (id < 0) {
            return id;
        }

        final PoolSubpage<T>[] subpages = this.subpages;
        final int pageSize = this.pageSize;
        freeBytes -= pageSize;
        // 定位 subPage，在 PoolChunk 中维护的 subPages[] 其实是叶子上对应的 page（如果对应位置作为了 subPage 的话）
        int subpageIdx = subpageIdx(id);
        PoolSubpage<T> subpage = subpages[subpageIdx];
        // 有可能为空，PoolChunk 中的 subPage 是接着 head 的 subPage
        // head 是 PoolArena 中 subpages 的头节点
        if (subpage == null) {
            // 包含了 init() 
            subpage = new PoolSubpage<T>(head, this, id, runOffset(id), pageSize, normCapacity);
            subpages[subpageIdx] = subpage;
        } else {
            // 重新初始化
            subpage.init(head, normCapacity);
        }
        return subpage.allocate();
    }
}
```
如何构建 `PoolSubPage`:
``` java
void init(PoolSubpage<T> head, int elemSize) {
    doNotDestroy = true;
    // 一份 subPage 的大小
    this.elemSize = elemSize;
    if (elemSize != 0) {
        // 最大/当前 可以使用的 subPage 个数
        // 如要开辟 16B 的子页，默认 pageSize = 8K，则为 512；如果开辟 4K 则为 2
        maxNumElems = numAvail = pageSize / elemSize;
        nextAvail = 0;
        bitmapLength = maxNumElems >>> 6;
        // 如果申请太大，切分数也最少为 2 (bitmapLength = 1)
        if ((maxNumElems & 63) != 0) {
            bitmapLength ++;
        }
        // 初始化 bitmap，标志着 page 的占用
        // bitMap 长度为 pageSize / 1024 默认 8，但不是所有位都有实际意义，有效位同 bitmapLength
        for (int i = 0; i < bitmapLength; i ++) {
            bitmap[i] = 0;
        }
    }
    // 加入队列，之后可以在 Arena 中进行引用了
    addToPool(head);
}
```

在 `PollArena` 中申请 `subPage` 内存空间，同样是 allocate 过程，简化一下上下文：

``` java
// 简化，以 tiny 为例
if (tiny) { // < 512
    // 尝试缓存
    if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
        // was able to allocate out of the cache so move on
        return;
    }
    tableIdx = tinyIdx(normCapacity);
    table = tinySubpagePools;
}
// tinyPool 为 512 >>> 4 = 32，smallPool 为 log2PageSize - 9 默认 4
// 这里省略 tinyIdx/smallIdx 的算法，tinyPool 被固定分成了 16/32/48...496 大小
// smallPool 则按指数形式 512/1024/2048/4096，8192 就不用 subPage 啦
final PoolSubpage<T> head = table[tableIdx];
synchronized (head) {
    final PoolSubpage<T> s = head.next;
    if (s != head) {
        assert s.doNotDestroy && s.elemSize == normCapacity;
        // 使用 subPage 进行内存申请
        long handle = s.allocate();
        assert handle >= 0;
        s.chunk.initBufWithSubpage(buf, null, handle, reqCapacity);
        incTinySmallAllocation(tiny);
        return;
    }
}
```

#### 总结 ####

`PooledByteBuf` 分配通过 `PooledByteBufAllocator` 执行。通过 `PoolThreadLocalCache` 做线程本地缓存，缓存 `PoolThreadCache` 对象。其中有不同大小的 `MemoryRegionCache[]`，内部通过 `Recycler` 进行对象回收利用。在缓存未命中的情况下，交给 `PoolArena` 进行内存申请。`PoolArena` 对不同大小的申请有两种策略，如果大于等于 *pageSize* 的交给 `PoolChunk` 进行分配，并由 `PollChunkList` 进行关联调整；如果小于 *pageSize*，在由 `PoolChunk` 初始化 `PoolSubPage` 之后，同过 `PoolSubPage` 来做小内存的分配。
- `PooledByteBufAllocator`
    - `PoolThreadLocalCache<PoolThreadCache>`
        - `MemoryRegionCache[]`
        - `PoolArena`
            - `PoolSubPage[]`
            - `PoolChunkList`
                - `PoolChunk`
