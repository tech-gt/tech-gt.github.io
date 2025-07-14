---
title: "Java实现一个简单的AQS，快速理解AQS原理"
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

在Java并发编程中，AQS（AbstractQueuedSynchronizer）是JUC包下同步器的核心基础。无论是ReentrantLock、CountDownLatch还是Semaphore，都是基于AQS实现的。很多同学在面试时能说出AQS的概念，但真正理解其设计思想和使用方法的人并不多。

今天我们就通过手写一个简化版的AQS，深入理解AQS的模版方法模式，看看它是如何优雅地解决同步问题的。

## 一、AQS的核心思想

AQS的设计非常巧妙，它采用了**模版方法模式**来实现同步器的框架：

1. **状态管理**：通过一个volatile的int类型state变量来表示同步状态
2. **队列管理**：通过一个FIFO的等待队列来管理获取锁失败的线程
3. **模版方法**：提供了一套模版方法，让子类只需要实现特定的方法即可

### AQS的核心组件

```java
public abstract class AbstractQueuedSynchronizer {
    private volatile int state;        // 同步状态
    private Node head;                 // 等待队列头节点
    private Node tail;                 // 等待队列尾节点
    
    // 需要子类实现的模版方法
    protected boolean tryAcquire(int arg) { throw new UnsupportedOperationException(); }
    protected boolean tryRelease(int arg) { throw new UnsupportedOperationException(); }
    protected int tryAcquireShared(int arg) { throw new UnsupportedOperationException(); }
    protected boolean tryReleaseShared(int arg) { throw new UnsupportedOperationException(); }
    protected boolean isHeldExclusively() { throw new UnsupportedOperationException(); }
}
```

## 二、AQS的模版方法详解

AQS通过模版方法模式，将复杂的同步逻辑分解成几个简单的模版方法，让子类只需要关注业务逻辑：

### 1. 独占锁相关的模版方法

```java
// 尝试获取独占锁
protected boolean tryAcquire(int arg)

// 尝试释放独占锁  
protected boolean tryRelease(int arg)

// 当前线程是否独占此锁
protected boolean isHeldExclusively()
```

### 2. 模版方法的职责

- `tryAcquire(int arg)`：尝试获取锁，成功返回true，失败返回false
- `tryRelease(int arg)`：尝试释放锁，完全释放返回true，否则返回false
- `isHeldExclusively()`：当前线程是否独占资源

AQS已经帮我们实现了复杂的队列管理、线程阻塞/唤醒等逻辑，我们只需要实现这几个简单的方法即可。

## 三、手写一个简化版AQS

为了更好地理解AQS的工作原理，我们来实现一个简化版的AQS：

```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.LockSupport;
import java.util.concurrent.ConcurrentLinkedQueue;

public abstract class SimpleAQS {
    // 同步状态
    private final AtomicInteger state = new AtomicInteger(0);
    
    // 等待队列
    private final ConcurrentLinkedQueue<Thread> waiters = new ConcurrentLinkedQueue<>();
    
    // 获取同步状态
    protected final int getState() {
        return state.get();
    }
    
    // 设置同步状态
    protected final void setState(int newState) {
        state.set(newState);
    }
    
    // CAS修改同步状态
    protected final boolean compareAndSetState(int expect, int update) {
        return state.compareAndSet(expect, update);
    }
    
    // 子类需要实现的模版方法
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
    
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
    
    protected boolean isHeldExclusively() {
        throw new UnsupportedOperationException();
    }
    
    // 获取锁（可能阻塞）
    public final void acquire(int arg) {
        if (!tryAcquire(arg)) {
            // 获取失败，加入等待队列
            waiters.add(Thread.currentThread());
            
            // 循环尝试获取锁
            while (!tryAcquire(arg)) {
                LockSupport.park(this);
            }
            
            // 获取成功，从等待队列中移除
            waiters.remove(Thread.currentThread());
        }
    }
    
    // 释放锁
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            // 释放成功，唤醒等待线程
            Thread next = waiters.poll();
            if (next != null) {
                LockSupport.unpark(next);
            }
            return true;
        }
        return false;
    }
}
```

## 四、基于SimpleAQS实现一个独占锁

现在我们来实现一个不可重入的独占锁，看看模版方法是如何工作的：

```java
public class SimpleExclusiveLock extends SimpleAQS {
    
    @Override
    protected boolean tryAcquire(int arg) {
        // 独占锁：state从0变为1表示获取成功
        return compareAndSetState(0, 1);
    }
    
    @Override
    protected boolean tryRelease(int arg) {
        // 释放锁：将state设为0
        setState(0);
        return true;  // 完全释放
    }
    
    @Override
    protected boolean isHeldExclusively() {
        return getState() == 1;
    }
    
    // 对外提供的接口
    public void lock() {
        acquire(1);
    }
    
    public void unlock() {
        release(1);
    }
    
    public boolean isLocked() {
        return isHeldExclusively();
    }
}
```

## 五、实现一个可重入锁

我们再来实现一个可重入锁，体会一下模版方法的灵活性：

```java
public class SimpleReentrantLock extends SimpleAQS {
    private Thread owner;  // 锁的持有者
    
    @Override
    protected boolean tryAcquire(int arg) {
        Thread current = Thread.currentThread();
        int c = getState();
        
        if (c == 0) {
            // 锁未被占用，尝试获取
            if (compareAndSetState(0, arg)) {
                owner = current;
                return true;
            }
        } else if (current == owner) {
            // 当前线程已持有锁，重入
            int nextc = c + arg;
            setState(nextc);
            return true;
        }
        
        return false;
    }
    
    @Override
    protected boolean tryRelease(int arg) {
        Thread current = Thread.currentThread();
        if (current != owner) {
            throw new IllegalMonitorStateException();
        }
        
        int c = getState() - arg;
        boolean free = false;
        
        if (c == 0) {
            // 完全释放锁
            free = true;
            owner = null;
        }
        
        setState(c);
        return free;
    }
    
    @Override
    protected boolean isHeldExclusively() {
        return owner == Thread.currentThread();
    }
    
    // 对外提供的接口
    public void lock() {
        acquire(1);
    }
    
    public void unlock() {
        release(1);
    }
    
    public boolean isLocked() {
        return getState() != 0;
    }
    
    public int getHoldCount() {
        return isHeldExclusively() ? getState() : 0;
    }
}
```

## 六、实际使用示例

让我们看看如何使用这些自定义的锁：

```java
public class LockDemo {
    public static void main(String[] args) throws InterruptedException {
        testSimpleReentrantLock();
    }
    
    
    // 测试可重入锁
    private static void testSimpleReentrantLock() throws InterruptedException {
        SimpleReentrantLock lock = new SimpleReentrantLock();
        
        Runnable task = () -> {
            String threadName = Thread.currentThread().getName();
            System.out.println(threadName + " 尝试获取锁");
            
            lock.lock();
            try {
                System.out.println(threadName + " 获取到锁，持有次数：" + lock.getHoldCount());
                
                // 重入
                lock.lock();
                try {
                    System.out.println(threadName + " 重入成功，持有次数：" + lock.getHoldCount());
                    Thread.sleep(1000);
                } finally {
                    lock.unlock();
                    System.out.println(threadName + " 释放一次锁，持有次数：" + lock.getHoldCount());
                }
                
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
                System.out.println(threadName + " 完全释放锁，持有次数：" + lock.getHoldCount());
            }
        };
        
        Thread t1 = new Thread(task, "线程A");
        Thread t2 = new Thread(task, "线程B");
        
        t1.start();
        t2.start();
        
        t1.join();
        t2.join();
        
        System.out.println("========== 可重入锁测试完成 ==========");
    }
}
```

## 七、运行结果

```
线程A 尝试获取锁
线程A 获取到锁，持有次数：1
线程A 重入成功，持有次数：2
线程B 尝试获取锁
线程A 释放一次锁，持有次数：1
线程A 完全释放锁，持有次数：0
线程B 获取到锁，持有次数：1
线程B 重入成功，持有次数：2
线程B 释放一次锁，持有次数：1
线程B 完全释放锁，持有次数：0
========== 可重入锁测试完成 ==========
```

## 八、AQS模版方法的优势

通过上面的例子，我们可以看到AQS模版方法模式的优势：

1. **关注点分离**：子类只需要关注业务逻辑（如何获取/释放锁），不需要关心复杂的队列管理
2. **代码复用**：所有同步器都可以复用AQS的队列管理、线程阻塞/唤醒等逻辑
3. **扩展性强**：通过重写不同的模版方法，可以实现各种不同的同步器
4. **性能优化**：AQS内部使用了大量的性能优化技巧，子类可以直接受益

## 九、总结

AQS的模版方法模式是一个经典的设计模式应用。它通过几个简单的模版方法，让我们可以轻松实现各种复杂的同步器。

在实际工作中，当你需要实现自定义的同步器时，不妨考虑使用AQS：
- 明确你的同步语义（独占还是共享）
- 定义好状态的含义
- 实现对应的模版方法
- 提供友好的对外接口

### 独占锁 vs 共享锁

**独占锁（Exclusive）**：同一时刻只能有一个线程持有锁
- 典型例子：ReentrantLock、synchronized
- 使用场景：临界区保护，确保同一时刻只有一个线程能访问共享资源
- 实现方式：重写`tryAcquire`和`tryRelease`方法

**共享锁（Shared）**：同一时刻可以有多个线程持有锁
- 典型例子：CountDownLatch（倒计时锁）、Semaphore（信号量）、ReadWriteLock的读锁
- 使用场景：允许多个线程同时访问，但需要控制访问数量或等待某个条件
- 实现方式：重写`tryAcquireShared`和`tryReleaseShared`方法

让我们通过一个具体的例子来理解：

```java
// 独占锁示例：停车场只有1个车位
public class ParkingLot extends SimpleAQS {
    public ParkingLot() {
        setState(1);  // 1个车位
    }
    
    @Override
    protected boolean tryAcquire(int arg) {
        // 独占：要么有车位（1），要么没有（0）
        return compareAndSetState(1, 0);
    }
    
    @Override
    protected boolean tryRelease(int arg) {
        setState(1);  // 释放车位
        return true;
    }
    
    public void park() { acquire(1); }
    public void leave() { release(1); }
}

// 共享锁示例：停车场有N个车位
public class MultiParkingLot extends SimpleAQS {
    public MultiParkingLot(int capacity) {
        setState(capacity);  // N个车位
    }
    
    @Override
    protected int tryAcquireShared(int arg) {
        for (;;) {
            int current = getState();
            int remaining = current - arg;
            if (remaining < 0) {
                return remaining;  // 没有足够车位
            }
            if (compareAndSetState(current, remaining)) {
                return remaining;  // 成功获取车位
            }
        }
    }
    
    @Override
    protected boolean tryReleaseShared(int arg) {
        for (;;) {
            int current = getState();
            int next = current + arg;
            if (compareAndSetState(current, next)) {
                return true;  // 成功释放车位
            }
        }
    }
    
    public void park() { acquireShared(1); }
    public void leave() { releaseShared(1); }
}
```

通过这个停车场的例子：
- **独占锁**：整个停车场只有1个车位，同一时刻只能停1辆车
- **共享锁**：停车场有N个车位，可以同时停N辆车，但不能超过容量

在选择同步语义时，关键是要明确：**你的资源是否允许多个线程同时访问？**
- 如果资源天然互斥（如银行账户余额修改），选择独占锁
- 如果资源可以被多个线程同时使用（如数据库连接池），选择共享锁

理解了AQS的模版方法，再去看JUC包下的各种同步器源码，你会发现它们都是这个套路，只是在具体的业务逻辑上有所不同。