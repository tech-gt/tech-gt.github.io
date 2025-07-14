---
title: "用Java实现一个简单的AQS，用于演示AQS原理"
date: 2024-06-08T10:00:00+08:00
categories:
  - Java并发
  - 源码剖析
  
tags:
  - AQS
  - 并发编程
  - Java
slug: aqs-demo
---

# 用Java实现一个简单的AQS，用于演示AQS原理

在日常Java并发编程中，AQS（AbstractQueuedSynchronizer）是JUC包下很多同步器（如ReentrantLock、CountDownLatch、Semaphore等）的核心基础。很多同学在面试或者实际开发中都听说过AQS，但真正理解其原理和实现机制的人并不多。本文将通过手写一个极简版AQS，帮助大家理解AQS的核心思想。

## 一、AQS原理简介

AQS的核心思想是：**通过一个int类型的state变量来表示同步状态，通过一个FIFO的等待队列来管理获取锁失败的线程**。当线程获取锁失败时，会被加入到等待队列中，等待唤醒。

AQS的设计极大地简化了同步器的开发，只需要继承AQS并实现tryAcquire/tryRelease等方法，就能快速实现自定义同步器。

## 二、为什么要自己实现一个简化版AQS？

源码虽好，但初学者直接看AQS源码容易迷失。自己实现一个简化版AQS，可以帮助我们抓住AQS的本质：
- 状态管理（state）
- 队列管理（等待线程）
- 线程的阻塞与唤醒

## 三、简化版AQS的Java实现

下面是一个极简版的AQS实现，仅支持独占锁：

```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.LockSupport;
import java.util.Queue;
import java.util.concurrent.ConcurrentLinkedQueue;

public class SimpleAQS {
    private final AtomicInteger state = new AtomicInteger(0);
    private final Queue<Thread> waiters = new ConcurrentLinkedQueue<>();

    // 尝试获取锁
    public boolean tryAcquire() {
        return state.compareAndSet(0, 1);
    }

    // 释放锁
    public void release() {
        state.set(0);
        Thread next = waiters.poll();
        if (next != null) {
            LockSupport.unpark(next);
        }
    }

    // 获取锁（不可重入）
    public void acquire() {
        if (!tryAcquire()) {
            waiters.add(Thread.currentThread());
            while (!tryAcquire()) {
                LockSupport.park();
            }
            waiters.remove(Thread.currentThread());
        }
    }
}
```

## 四、代码讲解

- `state`：用来表示锁的状态，0为未占用，1为已占用。
- `waiters`：等待队列，存放获取锁失败的线程。
- `tryAcquire()`：尝试获取锁，成功返回true。
- `acquire()`：获取锁，失败则进入等待队列并阻塞，直到被唤醒。
- `release()`：释放锁，并唤醒队列中的下一个线程。

这个实现没有考虑可重入、超时、中断等复杂情况，目的是突出AQS的核心思想。

## 五、应用场景举例

假如我们要实现一个自定义的独占锁，可以基于上面的SimpleAQS：

```java
public class SimpleLock {
    private final SimpleAQS aqs = new SimpleAQS();

    public void lock() {
        aqs.acquire();
    }

    public void unlock() {
        aqs.release();
    }
}
```

使用方法：
```java
SimpleLock lock = new SimpleLock();
lock.lock();
try {
    // 临界区代码
} finally {
    lock.unlock();
}
```

## 六、总结

AQS的本质就是用一个状态+一个队列来管理同步。理解了这个模型，再去看JUC包下的各种同步器源码就会豁然开朗。建议大家多动手实现、调试，才能真正掌握AQS的精髓。

如果你在实际开发中遇到并发相关的问题，不妨试着用AQS的思想去分析和解决，说不定会有新的收获！ 