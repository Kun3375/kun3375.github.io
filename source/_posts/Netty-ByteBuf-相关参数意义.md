---
title: Netty ByteBuf 相关参数意义
author: 斯特拉不用电
date: 2020-02-24 16:52:18
tags: 
    - Netty
    - Java
categories: Java
comments: true
---

这里仅记录一下 `ByteBuf` 相关的一下内存配置参数和默认情况，分析一下 Allocator 的组成，为分析 `ByteBuf` 做一些概念上的铺垫。

首先看几个 `PooledByteBifAllocator` 中的静态变量，挑了几个说明一下：
``` java
// 默认的 arena 数量，一个对应 HeapBuf 一个对应 DirectBuf
private static final int DEFAULT_NUM_HEAP_ARENA;
private static final int DEFAULT_NUM_DIRECT_ARENA;
// 默认 pageSzie、order 默认的 chunk = pageSize << order
private static final int DEFAULT_PAGE_SIZE;
private static final int DEFAULT_MAX_ORDER;
// 三个不同大小的 cacheSize
private static final int DEFAULT_TINY_CACHE_SIZE;
private static final int DEFAULT_SMALL_CACHE_SIZE;
private static final int DEFAULT_NORMAL_CACHE_SIZE;
```

再来看一下 `PooledByteBufAllocator` 的静态代码块，主要是为了结合机器环境和设置值对上面的静态变量进行初始化：

``` java
static {
    // 设置页大小，默认 8KB
    int defaultPageSize = SystemPropertyUtil.getInt("io.netty.allocator.pageSize", 8192);
    Throwable pageSizeFallbackCause = null;
    try {
        // 校验页大小是否是 2 的幂，且大于最小值
        validateAndCalculatePageShifts(defaultPageSize);
    } catch (Throwable t) {
        // 如果设置的页大小有问题则重新设置为 8KB
        pageSizeFallbackCause = t;
        defaultPageSize = 8192;
    }
    DEFAULT_PAGE_SIZE = defaultPageSize;

    // 同样的设置 MaxOrder，默认 11。chunk = page << order 即要申请的内存空间
    int defaultMaxOrder = SystemPropertyUtil.getInt("io.netty.allocator.maxOrder", 11);
    Throwable maxOrderFallbackCause = null;
    try {
        // 校验 MaxOrder 在 0~14 间，并且块大小不超过最大限制
        validateAndCalculateChunkSize(DEFAULT_PAGE_SIZE, defaultMaxOrder);
    } catch (Throwable t) {
        // 检验失败则重置为默认的 11
        maxOrderFallbackCause = t;
        defaultMaxOrder = 11;
    }
    DEFAULT_MAX_ORDER = defaultMaxOrder;


    final Runtime runtime = Runtime.getRuntime();

    // 有一段官方说明，默认使用处理器数量两倍的 Arena（这和 Epoll/Nio EventLoop 的数量一致来避免争用）所以一般如果需要修改 EventLoop 的线程数量，需要同时定制好 ByteBuf 的分配器。
    // 默认 Arena 最小数量为处理器核心数两倍
    final int defaultMinNumArena = NettyRuntime.availableProcessors() * 2;
    // 默认 Chunk = PageSize << MaxOrder 按默认配置就是 16MB
    final int defaultChunkSize = DEFAULT_PAGE_SIZE << DEFAULT_MAX_ORDER;
    // 默认 Arena 数量取决于以下两者的最小值
    // - 处理器核心数两倍
    // - 对内存占用不超过 50% 且保证每个 Arena 拥有 3 个 Chunk
    // 默认的一个 Arena 16*3=48MB 
    DEFAULT_NUM_HEAP_ARENA = Math.max(0,
            SystemPropertyUtil.getInt(
                    "io.netty.allocator.numHeapArenas",
                    (int) Math.min(
                            defaultMinNumArena,
                            runtime.maxMemory() / defaultChunkSize / 2 / 3)));
    // 对直接内存也类似处理
    DEFAULT_NUM_DIRECT_ARENA = Math.max(0,
            SystemPropertyUtil.getInt(
                    "io.netty.allocator.numDirectArenas",
                    (int) Math.min(
                            defaultMinNumArena,
                            PlatformDependent.maxDirectMemory() / defaultChunkSize / 2 / 3)));

    // 初始化三个 size 
    DEFAULT_TINY_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.tinyCacheSize", 512);
    DEFAULT_SMALL_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.smallCacheSize", 256);
    DEFAULT_NORMAL_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.normalCacheSize", 64);

    // 缓存 Buffer 的默认最大大小
    DEFAULT_MAX_CACHED_BUFFER_CAPACITY = SystemPropertyUtil.getInt(
            "io.netty.allocator.maxCachedBufferCapacity", 32 * 1024);

    // 设定超过多少大小进行缓存释放
    DEFAULT_CACHE_TRIM_INTERVAL = SystemPropertyUtil.getInt(
            "io.netty.allocator.cacheTrimInterval", 8192);
    // 设定超过多少时间进行缓存释放，默认仅依靠数量阈值释放缓存
    DEFAULT_CACHE_TRIM_INTERVAL_MILLIS = SystemPropertyUtil.getLong(
            "io.netty.allocation.cacheTrimIntervalMillis", 0);
    // 是否允许所有线程访问缓存内容
    DEFAULT_USE_CACHE_FOR_ALL_THREADS = SystemPropertyUtil.getBoolean(
            "io.netty.allocator.useCacheForAllThreads", true);
    // 是否内存对齐，默认关闭
    DEFAULT_DIRECT_MEMORY_CACHE_ALIGNMENT = SystemPropertyUtil.getInt(
            "io.netty.allocator.directMemoryCacheAlignment", 0);

    // 设置每个 Chunk 可以缓存多少 ByteBuffer
    DEFAULT_MAX_CACHED_BYTEBUFFERS_PER_CHUNK = SystemPropertyUtil.getInt(
            "io.netty.allocator.maxCachedByteBuffersPerChunk", 1023);

    // 省略 Debug Log
}
```
---

静态变量作为分配器的一些阈值、默认值，那么这些成员变量控制着分配器实际状态

``` java
// 这里的成员变量都会在下面的全参构造器中进行初始化
private final PoolArena<byte[]>[] heapArenas;
private final PoolArena<ByteBuffer>[] directArenas;
private final int tinyCacheSize;
private final int smallCacheSize;
private final int normalCacheSize;
private final List<PoolArenaMetric> heapArenaMetrics;
private final List<PoolArenaMetric> directArenaMetrics;
// 这是分配器的核心，缓存池。在完成分配器的初始化后会分析
private final PoolThreadLocalCache threadCache;
private final int chunkSize;
private final PooledByteBufAllocatorMetric metric;
```

再看 `PooledByteBufAllocator` 的全参构造器，包含了缓存池的初始化：

``` java
public PooledByteBufAllocator(boolean preferDirect, int nHeapArena, int nDirectArena, int pageSize, int maxOrder,
                                  int tinyCacheSize, int smallCacheSize, int normalCacheSize,
                                  boolean useCacheForAllThreads, int directMemoryCacheAlignment) {
    // 确认了是否直接内存优先
    super(preferDirect);
    // PoolThreadLocalCache 是 Netty FastThreadLocal 子类，Java ThreadLocal 更快
    threadCache = new PoolThreadLocalCache(useCacheForAllThreads);
    // 三个 cacheSize 大小赋值、chunkSize 赋值
    this.tinyCacheSize = tinyCacheSize;
    this.smallCacheSize = smallCacheSize;
    this.normalCacheSize = normalCacheSize;
    chunkSize = validateAndCalculateChunkSize(pageSize, maxOrder);
    // 验证 Arena 数量
    checkPositiveOrZero(nHeapArena, "nHeapArena");
    checkPositiveOrZero(nDirectArena, "nDirectArena");
    // Alignment 对齐参数验证
    checkPositiveOrZero(directMemoryCacheAlignment, "directMemoryCacheAlignment");
    // 内存对齐需要 unsafe 的支持，判断如果 JDK 实现中没有的话是不支持内存对齐的
    if (directMemoryCacheAlignment > 0 && !isDirectMemoryCacheAlignmentSupported()) {
        throw new IllegalArgumentException("directMemoryCacheAlignment is not supported");
    }
    // 继续验证对齐值是否为 2 的幂次方
    if ((directMemoryCacheAlignment & -directMemoryCacheAlignment) != directMemoryCacheAlignment) {
        throw new IllegalArgumentException("directMemoryCacheAlignment: "
                + directMemoryCacheAlignment + " (expected: power of two)");
    }
    // pageShift = log2(pageSize)
    int pageShifts = validateAndCalculatePageShifts(pageSize);
    // 如果内存不够是无法开启 arena 的，而 arena 是池化的核心之一
    if (nHeapArena > 0) {
        // 初始化 arena，这些 arena 会被缓存池 PoolThreadLocalCache 引用
        heapArenas = newArenaArray(nHeapArena);
        List<PoolArenaMetric> metrics = new ArrayList<PoolArenaMetric>(heapArenas.length);
        for (int i = 0; i < heapArenas.length; i ++) {
            PoolArena.HeapArena arena = new PoolArena.HeapArena(this,
                    pageSize, maxOrder, pageShifts, chunkSize,
                    directMemoryCacheAlignment);
            heapArenas[i] = arena;
            metrics.add(arena);
        }
        heapArenaMetrics = Collections.unmodifiableList(metrics);
    } else {
        heapArenas = null;
        heapArenaMetrics = Collections.emptyList();
    }
    
    // 省略 directArena 初始化部分，同上
    
    // 对 metric 初始化，暴露该分配器的的一些信息
    metric = new PooledByteBufAllocatorMetric(this);
}
```
到这里分配器的本身的初始化已经完成，包含了一个缓存池：`PoolThreadLocalCache`。这是 `FastThreadLocal<PoolThreadCache>` 的子类，同时也是 `PooledByteBufAllocator` 的一个非静态内部类，缓存池内的很多信息直接引用了分配器的一些成员变量，后续的 `ByteBuf` 的分配都是直接或者间接地通过这个缓存来执行的。

