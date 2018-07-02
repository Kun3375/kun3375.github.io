---
title: AQS 浅析
date: 2018-07-02 11:27:16
tags: 
  - Java
  - lock
categories: Java
---
`java.util.concurrent` 包中提供了许多易用而有效的工具类，诸如 **ReentrantLock**，**CountDownLatch**，**Semaphore** 等等，给日常的并发编程带来了许多便利。他们都使用了同样的框架来完成基本的同步过程：***AbstractQueuedSynchronizer*** （AQS）来实现基本功能，比如获取资源和释放的步骤。

<!-- more -->

## 简单了解 ##

戳开 AQS 一览其结构，其实它本身维护了两个概念：
- state：（volatile int）该属性维护了资源的状态，或者是数量。
- CLH queue：一个先进先出的线程等待队列。这并不是一个具体对象，而是通过内部类 Node 来维护的。

AQS 对于 state 支持两种模式的资源共享形式：
- Exclusive-排他式：进程独占形式，如 ReentrantLock，Mutex 的应用。
- Share-共享式：支持多线程访问，典型的应用式 CountDownLatch / Semaphore。

子类实现 AQS 时候只需要实现关于 state 的获取（acquire）和释放（release）方案即可，包括队列的维护，线程的唤醒等工作，大部分都在 AQS 维护了。举个栗子，在使用 CountDownLatch 的时候，我们会初始化一个计数值 n 用于对应 n 个子线程，这个 n 同时也对应了 state 值，每一次 `countDown()` 的时候，会使得 state CAS 地减 1。在 state 归零的时候会使用 `unpark()`，主线程 从 `await()` 函数返回。

## 工作流程 ##

AQS 的 API 同一般的同步工具 API 一样，除了对于资源的 `acquire` / `release` 操作，还提供的了 `tryAcquire` / `tryRelease` 的非阻塞操作。同时 `acquireInterruptibly` 支持线程中断。如果需要使用共享式的操作，需要实现对应的 Share 操作方法。

### 资源获取 ###

首先看 `acquire` 方法：
``` java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

- `tryAcquire(int)`：尝试获取资源
- `addWaiter(Node)`：使线程（Node 对象维护这线程对象）进入等待队列，对于 acquire 方法使用 EXCLUSICE-排他模式。
- `acquireQueued(Node, int)`：在这一步骤中线程会等待，而在线程唤醒后会尝试在该方法内获取资源。
- `selfInterrupt`：由于线程在等待过程中无法响应中断，所以在获取资源并退出队列后补充一个中断。

`tryAcquire(int)` 方法默认会抛出操作不支持的异常，需要子类的具体实现。

``` java
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

`addWaiter(Node, int)` 方法会自旋使用 CAS 方式将一个 Node 加入队尾。

``` java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

从上面两个方法可以看出位于队列头部的 Node 其实只是一个标记，在队列第二位置的时候，线程已经可以获取资源并进行相关任务了。

`acquireQueued(Node, int)` 更关键的一步，在获取资源失败， Node 已经被加入队尾之后，线程需要进入等待状态等待被唤醒。

``` java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

shouldParkAfterFailedAcquire(Node, Node) 每个节点是否需要等待需要阻塞取决于前驱节点的状态。

``` java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

前驱节点最后会通过该状态值来判断是否需要 unpark 下个线程。在这里，如果前驱节点标识为 SIGNAL，则进入等待；标识为 CANCAL 需要追溯更前的节点状态；如果为其他正常值，则更新为 SIGNAL。

`parkAndCheckInterrupt()` 使线程进入 WATING，等待 unpark（在 release 中触发，马上就来） 或者 interrupt，被唤醒后回到 `acquireQueued` 触发中断或者继续检查是否可以获取资源。

``` java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

### 释放资源并唤醒后置线程 ###

这是 `acquire` 方法的反操作，用于资源的释放，当资源成功释放时，唤醒下一个线程（位于头节点之后）。

``` java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

通过 `tryRelease(int)` 方法判断资源是否释放，这个方法同样需要被实现：

``` java
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
```

而 `unparkSuccessor(Node)` 方法用来真正唤醒 node.next 中的线程：

``` java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

看到这就能了解整个 AQS 的运作流程了。在 parkAndCheckInterrupt 中进入 WAITING 的线程，在这里被唤醒，它会继续进入 `acquireQueued` 中的自旋，如果 `tryAcquire` 顺利获得资源，则将本线程的节点设置为 head 并返回 acquire 方法。so, go on.

AQS 本身并不复杂，使用时只需要手动实现 `tryAcquire` 和 `tryRealeas` 方法。

而对于 Share-共享式的 acquire / release 流程，区别并不太大，有兴趣的小伙伴可以自行翻阅源码一探究竟。

###### 参考 ######
1. <https://mp.weixin.qq.com/s/eyZyzk8ZzjwzZYN4a4H5YA>