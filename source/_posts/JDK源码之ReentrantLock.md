---
title: JDK源码之ReentrantLock
date: 2019-08-22 21:39:38
tags: Concurrency
categories: 技术
---

## 为什么要引入 ReentrantLock

Java5 之前已经有了内置锁：synchronized，为什么还要引入 ReentrantLock 呢？

原因有以下几点：

1. 使用内置锁无法中断一个正在等待获取锁的线程，或者无法在请求获取一个锁时无限地等待下去。
2. 内置锁无法实现非阻塞结构的加锁机制。



## 什么是 ReentrantLock

在Java5中引入了一个新的接口 Lock，其方法如下：

<div align=center>![ReentrantLock](./lock_interface.png)

从接口方法中可以看出：该锁实现了可中断、轮询锁、定时锁等功能。

ReentrantLock 实现了 Lock 接口，并**提供了与 synchronized 相同的互斥性和内存可见性**，此外，还提供了可重入的加锁语义。

也就是说在同一时刻，只能有一个线程持有该锁，且该线程可以重复持有。

## 实现

### 公平锁和非公平锁

公平锁意味着：线程将按照它们发出请求的顺序获取锁。

如果线程A获取锁失败，那么该线程将排到队尾，直到队列前面的线程都获取并释放锁后，才能轮到它获取锁。

非公平锁意味着：线程并不“按套路出牌”，如果获取锁失败，则可以“插队”再去获取锁。

```Java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

在其构造函数中，fair 参数代表是否公平。

在 ReentrantLock 有一个内部类 Sync，该内部类派生两个子类：FairSync 和 NonfairSync，分别代表了公平锁和非公平锁。

```Java
public void lock() {
    sync.lock();
}
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}
public boolean tryLock() {
    return sync.nonfairTryAcquire(1);
}
public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}
public void unlock() {
    sync.release(1);
}
```

可以看到，在 ReentrantLock 里的所有主要方法的实现，都全权委托 sync 实现了，而 Sync 类又是从 AbstractQueuedSynchronizer 继承过来的，所以如果想分析 ReentrantLock 的加锁机制，AbstractQueuedSynchronizer  是绕不开的。事实上，juc 里面的绝大多数类的实现都是以 AbstractQueuedSynchronizer  为基础的。

首先看下非公平锁如何实现。

```Java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

CAS 算是并发包里面的常用手段了。简言之，是一个原子操作，其接受两个参数：expect（期待旧值） 和 update（更新新值）。

假设变量的当前值为: now.

这个操作使用伪代码就可以描述如下：

> if now == expect
>
> ​	now = update

虽然这是一个判断语句再加一个更新语句，但是**这是一个原子操作**。

在这段代码中的逻辑是这样的：

**如果 state 变量的值是 0，则更新为 1，并设置该 lock 的归属线程是当前线程；否则去执行获取锁操作。**

显然，在  AbstractQueuedSynchronizer  中 `state` 变量值为 `0` 代表当前没有线程占有锁，`state` 变量为 `1` 代表已有线程占有该锁。

再看看如何通过非公平的方式获取该锁的：

```Java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    //暂时无线程占有该锁
    if (c == 0) {
        //传入的acquires参数的值为：1
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //有线程占有该锁，判断占有该锁的线程是不是当前线程，以此来实现“重入”功能
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

当然，在实际的实现中，如果此时还是没有抢到该锁，则该线程会排到等待队列中再次等待获取锁，这涉及到了AbstractQueuedSynchronizer  类的实现机制，此处为了脉络清晰，暂时放过不表。

现在可以看看公平锁的实现方式了。

```Java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    //如果当前没有线程占有该锁
    if (c == 0) {
        //前面没有排队等待的线程
        if (!hasQueuedPredecessors() &&
            //传入的 acquires 参数值为1
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

释放锁的过程与之正好相反：

```Java
protected final boolean tryRelease(int releases) {
    //releases 传入的值是 1
    int c = getState() - releases;
    //如果当前占有锁的线程不是当前线程，是无法执行释放锁的操作的，这也保证了线程安全
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        //清空占有锁的线程
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

### 小结

想想挺神奇，在没有使用任何与JVM相关内建的关键字的情况下，纯属用 Java 的类和一个简单的 CAS 操作就能实现各种各样的锁，不得不服。

说白了，加锁和释放锁都是通过更改 AQS 中的 `state` 变量实现的， state 为 `1`，有线程占有锁，state 为 `0`，没有线程占有锁。

## 如何在 ReentrantLock 与 synchronized 之间选择

1. synchronized 关键字简洁紧凑，ReentrantLock 的危险性较高，如果忘记在 finally 中释放锁将导致不可预料的错误；
2. 在一些内置锁无法满足需求的情况下，ReentrantLock可以作为一种高级工具。当需要一些高级功能时才应该使用 ReentrantLock，这些功能包括：可轮询、可定时与可中断。或者公平队列，以及一些非块结构的锁。
3. synchronized 是 JVM 的内置属性，JVM执行一些语句优化，而对 ReentrantLock 是不可知的。

（全文完）

## 参考资料

1. 《Java并发编程实战》