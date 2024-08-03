---
layout: post
title: Java ReentrantLock原理
date: 2024-04-28 19:50:00 +0800
categories: Java
author: Lundrea
tags:
- Java  
---

**_ReentrantLock_** 是 _Java JUC(java.util.concurrent)_ 包下的一个锁工具，它实现了 _Lock_ 接口，与 _synchronized_ 锁不同，**_ReentrantLock_** 除了用来做线程间互斥之外，还提供了很多高级的特性，例如**公平锁 & 非公平锁**以及**可中断**。  

本文将从 **JDK17** 源码角度介绍一下 **_ReentrantLock_** 的底层实现原理，这部分是 _Java_ 面试的常考知识点。（目前看到的博客都是基于 **JDK8** 源码分析的，而鲜有基于 **JDK17** 源码进行分析的）  

_2024美团暑期实习后端开发一面：公平锁与非公平锁是如何实现的？_ 看完这篇文章你就明白了！








## AQS

**AQS** 的全称为 _AbstractQueueSynchronizer_ ，即抽象队列同步器，通俗来讲，**AQS** 的作用就是来定义线程如何获得锁、线程未获得锁如何等待以及线程如何释放锁。**_ReentrantLock_** 的底层实现是高度依赖 **AQS** 的，这一点从源码中就可以看得出来：

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = 7373984872572414699L;
    /** Synchronizer providing all implementation mechanics */
    private final Sync sync;

    /**
     * Base of synchronization control for this lock. Subclassed
     * into fair and nonfair versions below. Uses AQS state to
     * represent the number of holds on the lock.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        ···
    }
    ···
}
```
**_ReentrantLock_** 内部有一个 _Sync_ 类型的对象，_Sync_ 这个类则是继承自 _AbstractQueuedSynchronizer_，也就是抽象队列同步器，这个锁的同步控制基础就是这个 _Sync_ 提供的，可以注意到这里的 _Sync_ 仍然是抽象类，其实它还有两个子类，分别是 _FairSync_ 和 _NonFairSync_ ，分别对应于公平锁的实现和非公平锁的实现。  

**AQS** 底层是一个 _CLH_ 队列，是一个双向队列，由 _Node_ 节点连接而成，这部分代码位于 _AbstractQueuedSynchronizer_ 抽象类中，

```java
/** CLH Nodes */
abstract static class Node {
    volatile Node prev;       // initially attached via casTail
    volatile Node next;       // visibly nonnull when signallable
    Thread waiter;            // visibly nonnull when enqueued
    volatile int status;      // written by owner, atomic bit ops by others
    
    ···
}
```

可以看到，一个 _Node_ 包含指向前后 _Node_ 的指针(prev 和 next)，一个在等待的线程对象(waiter)以及状态(status)。这里的状态指的是这个线程对象的状态，在 **JDK17** 中，这个状态只有三种，但在 **JDK8** 中这里的状态数量更多。  

```java
// Node status bits, also used as argument and return values
static final int WAITING   = 1;          // must be 1
static final int CANCELLED = 0x80000000; // must be negative
static final int COND      = 2;          // in a condition wait
```

其中 **WAITING** 表示线程在等待，**CANCELLED** 表示线程已取消获取锁(用于实现锁的可中断)，**COND** 表示线程处于一个条件等待的状态。  

除了 _CLH_ 队列外，**AQS** 中还有一个状态字段，即：

```java
/**
 * The synchronization state.
*/
private volatile int state;
```

用于标识锁目前是否被占用，等于 0 则锁空闲，大于 0 则锁被占用。  

一个获取锁失败的线程会被封装成为一个 _Node_ 对象放入到队列中排队等待获取锁。  

知道了 **AQS** 的概念，现在我们可以通过 **_ReentrantLock_** 加锁的全流程来窥探到 **_ReentrantLock_** 的底层原理了。  

## ReentrantLock 加锁流程

首先是实例化一个 **_ReentrantLock_** 锁对象，这里构造函数可带参数，如果参数为 _True_ ，则会实例化一个公平锁，默认为非公平锁，其实区别就在于是实例化了一个 _FairSync_ 对象还是一个 _NonFairSync_ 对象。

```java
// 实例化一个锁
Lock lock = new ReentrantLock();

// 构造函数的实现
// 非公平锁
public ReentrantLock() {
    sync = new NonfairSync();
}
// 公平锁
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

之后调用 `lock()` 方法加锁，其实底层是调用了 _sync_ 对象的 `lock()` 方法来加锁，看到这里，你可能更能理解为什么说 **_ReentrantLock_** 的底层实现是高度依赖 **AQS** 的。  

```java
public void lock() {
    sync.lock();
}
```

`sync.lock()` 的实现如下：

```java
final void lock() {
    if (!initialTryLock())
        acquire(1);
}
```

这里 `initialTryLock()` 方法是一个抽象方法，实际上调用的是 _Sync_ 这个类的两个子类 _FairSync_ 和 _NonFairSync_ 的重写的 `initialTryLock()` 方法。  

我们先看非公平锁 _FairSync_ 的实现：

```java
/**
* Sync object for non-fair locks
*/
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    final boolean initialTryLock() {
        Thread current = Thread.currentThread();
        if (compareAndSetState(0, 1)) { // first attempt is unguarded
            setExclusiveOwnerThread(current);
            return true;
        } else if (getExclusiveOwnerThread() == current) {
            int c = getState() + 1;
            if (c < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(c);
            return true;
        } else
            return false;
    }

    /**
     * Acquire for non-reentrant cases after initialTryLock prescreen
     */
    protected final boolean tryAcquire(int acquires) {
        if (getState() == 0 && compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }
}
```

非公平锁的 `initialTryLock()` 方法通过 _CAS_ 尝试加锁(即 `compareAndSetState(0, 1)`)，如果 **AQS** 的 _state_ 字段为 0 ，则把这个字段变为 1 ，并使用 `setExclusiveOwnerThread(current);` 将独占锁的线程设置为当前线程即自己，否则加锁失败，失败之后会判断当前独占锁的线程是不是当前线程。如果是的话将 _state_ 加一，同样加锁成功，这里就体现了可重入锁的实现，即同一个线程可以多次获取锁而不阻塞，_state_ 字段其实就是这个线程获取锁的次数。如果这两种情况都没有加锁成功，则认为锁被另一个线程独占，加锁失败，返回 _false_ 。  

在第一次 _CAS_ 加锁未成功时，会调用 `acquire(1)` 这个方法，这个方法是 _AbstractQueuedSynchronizer_ 抽象类提供给我们的，接下来我们看看这个方法的实现。  

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg))
        acquire(null, arg, false, false, false, 0L);
}
```

首先调用 `tryAcquire(arg)` 方法，这里会去调用具体实现类的 `tryAcquire()` 方法，同样，我们先看非公平锁的实现，实际上上面已经给出，这里只摘出 `tryAcquire()` 的实现：

```java
protected final boolean tryAcquire(int acquires) {
    if (getState() == 0 && compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(Thread.currentThread());
        return true;
    }
    return false;
}
```

这里是尝试了第二次 _CAS_ 加锁，与 `initialTryLock()` 方法中的逻辑差不多，同样也是做 _CAS_ 尝试，如果成功，将独占这个锁的线程设置为自己，如果不成功则返回 _false_ 。  

如果第二次 _CAS_ 加锁不成功，则调用 `acquire(null, arg, false, false, false, 0L)` 方法，这个方法极其重要，它的内部定义了线程如何排队来获取锁的逻辑。  

下面是这个方法的完整实现，我添加了较为详细的注释便于理解：

```java
final int acquire(Node node, int arg, boolean shared,
                  boolean interruptible, boolean timed, long time) {
    // node 是将要加入 DLH 队列的节点
    // spins 是队头节点在竞争锁时可以自旋的次数
    // interrupted 当前线程是否中断获取锁
    // first 当前线程是否位于队列第一个节点中
    // pred 当前节点的前一个节点
    Thread current = Thread.currentThread();
    byte spins = 0, postSpins = 0;   // retries upon unpark of first thread
    boolean interrupted = false, first = false;
    Node pred = null;                // predecessor of node when enqueued

    /*
     * Repeatedly:
     *  Check if node now first
     *    if so, ensure head stable, else ensure valid predecessor
     *  if node is first or not yet enqueued, try acquiring
     *  else if node not yet created, create it
     *  else if not yet enqueued, try once to enqueue
     *  else if woken from park, retry (up to postSpins times)
     *  else if WAITING status not set, set and retry
     *  else park and clear WAITING status, and check cancellation
     */
    
    // 一个大循环，来控制线程对锁的竞争
    for (;;) {
        // 如果当前节点不为第一个节点、前一个节点也不为空、头结点也不等于前一个节点
        if (!first && (pred = (node == null) ? null : node.prev) != null &&
            !(first = (head == pred))) {
            // 如果前一个节点状态小于 0，说明前驱是一个取消了获取锁请求的线程
            // 则清理队列，去除掉取消获取锁的节点
            if (pred.status < 0) {
                cleanQueue();           // predecessor cancelled
                continue;
            // 如果前驱的前驱为空，
            } else if (pred.prev == null) {
                // 这是一个自旋等待提示，告诉CPU当前线程正在自旋等待，
                // 处理器可以进行一些针对自旋的优化
                Thread.onSpinWait();    // ensure serialization
                continue;
            }
        }
        // 如果当前节点是第一个节点或者前驱为空，则竞争锁
        if (first || pred == null) {
            boolean acquired;
            try {
                if (shared)
                    acquired = (tryAcquireShared(arg) >= 0);
                else
                    // CAS 获取锁
                    acquired = tryAcquire(arg);
            } catch (Throwable ex) {
                cancelAcquire(node, interrupted, false);
                throw ex;
            }
            // 如果成功获取到了锁
            if (acquired) {
                // 如果还是第一个，把这个节点从队列中拿下来，赋值给 head
                if (first) {
                    node.prev = null;
                    head = node;
                    pred.next = null;
                    node.waiter = null;
                    if (shared)
                        signalNextIfShared(node);
                    if (interrupted)
                        current.interrupt();
                }
                return 1;
            }
        }
        // 如果当前节点为空
        if (node == null) {                 // allocate; retry before enqueue
            if (shared)
                node = new SharedNode();
            else
                // 创建节点
                node = new ExclusiveNode();
        // 如果前驱为空，入队
        } else if (pred == null) {          // try to enqueue
            node.waiter = current;
            Node t = tail;
            node.setPrevRelaxed(t);         // avoid unnecessary fence
            if (t == null)
                tryInitializeHead();
            else if (!casTail(t, node))
                node.setPrevRelaxed(null);  // back out
            else
                t.next = node;
        // 如果是第一个节点且自旋次数不为 0 自旋次数减，且调用 onSpinWait()
        } else if (first && spins != 0) {
            --spins;                        // reduce unfairness on rewaits
            Thread.onSpinWait();
        // 如果当前节点状态为 0 ，说明是新节点，初始化状态为 WAITTING
        } else if (node.status == 0) {
            node.status = WAITING;          // enable signal and recheck
        // 重新计算自旋次数，并且由于当前线程自旋次数用完，被 park 住
        } else {
            long nanos;
            spins = postSpins = (byte)((postSpins << 1) | 1);
            if (!timed)
                LockSupport.park(this);
            else if ((nanos = time - System.nanoTime()) > 0L)
                LockSupport.parkNanos(this, nanos);
            else
                break;
            node.clearStatus();
            if ((interrupted |= Thread.interrupted()) && interruptible)
                break;
        }
    }
    return cancelAcquire(node, interrupted, interruptible);
}
```

这段核心代码的作用在于控制未获得锁的线程入队，同时让队列头部的线程参与争夺锁。需要注意的是，这里的 _head_ 指向抢到锁的线程对应的 _node_。  

首先第一个 _if_ 中，
1. 如果发现有取消获取锁的线程，会执行队列清理，
2. 如果自己是第二个节点，那么会一直自旋来判断前一个线程是否放弃了获取锁，防止队头线程已经放弃了获取锁，而在队列之后的线程还不知道，造成饥饿。  

第二个 _if_ 中，
1. 如果当前节点是队列中的第一个节点或者前驱为空，则参与争夺锁，调用了 `tryAcquire()` 方法，也就是说争夺锁的线程只有队列中的第一个 _Node_ 中的线程、刚释放了锁的那个线程以及还尚未进入这个函数的线程（在进入这个函数之前会尝试两次 _CAS_ 来获取锁）。  

第三个 _if_ 中，
1. 如果当前节点还为空，则实例化一个 _node_，
2. 如果前驱为空，则入队，
3. 如果当前节点为队列中第一个节点，且自旋次数(spins)未用尽，则 -1 ，之后接着做下一次自旋
4. 如果当前节点状态为 0 ，说明是新节点，将状态设置为 WAITTING
5. 最后 _else_ 中会先计算当前线程的自旋次数 _spins_，并将当前线程 **park** 住  

按照源码中 _spins_ 的计算方法，每当 _spins_ 用尽，下一次的自旋次数为上一次的自旋次数的 2 倍 +1 。  

可以看到，队列中第一个 _node_ 中的线程等待的时间越长，其可以自旋的次数就会越多，这样可以有效地避免队列中的线程饥饿，因为自旋次数越多，获取锁的概率越大，所以随着等待时间的增加，获取锁的概率其实也是增加的。**JDK17**中是这样的，而在 **JDK8** 中却有所不同，**JDK8** 中线程每次被唤醒只会自旋一次就迅速被 _park_ 住，这样就比较容易造成饥饿。  

概括来讲整个加锁过程，一个线程首先会自旋两次尝试获取锁，如果这两次都没有获取到锁，则进入到 `acquire()` 方法的大循环中，队列中的所有在等待的线程都会被 _park_ 住, 直到第一个线程被唤醒，然后开始自旋，直到自旋次数用尽，被再次 _park_ 住......  

你可能会问，那么谁来唤醒队列中第一个等待的线程呢？当然是刚刚释放锁的那个线程啦~  

解锁也是调用了 _sync_ 对象的 `release()` 方法，下面是解锁部分的源码：  

```java
public void unlock() {
    sync.release(1);
}

public final boolean release(int arg) {
    if (tryRelease(arg)) {
        signalNext(head);
        return true;
    }
    return false;
}

protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (getExclusiveOwnerThread() != Thread.currentThread())
        throw new IllegalMonitorStateException();
    boolean free = (c == 0);
    if (free)
        setExclusiveOwnerThread(null);
    setState(c);
    return free;
}

private static void signalNext(Node h) {
    Node s;
    if (h != null && (s = h.next) != null && s.status != 0) {
        s.getAndUnsetStatus(WAITING);
        LockSupport.unpark(s.waiter);
    }
}
```

`tryRelease()` 方法尝试解锁，如果成功解锁，则调用 `signalNext()` 方法，唤醒队列中的第一个线程，于是队列中的第一个线程就加入到了获取锁的行列中来。  

这就是 **_ReentrantLock_** 加锁过程及其底层原理，你可能会疑惑为什么前面都在讲非公平锁，那公平锁呢？  

## 公平锁  

其实公平锁的实现与非公平锁及其类似，只有一处不同，具体可以看公平锁的源码：

```java
/**
* Sync object for fair locks
*/
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    /**
     * Acquires only if reentrant or queue is empty.
     */
    final boolean initialTryLock() {
        Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedThreads() && compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        } else if (getExclusiveOwnerThread() == current) {
            if (++c < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(c);
            return true;
        }
        return false;
    }

    /**
     * Acquires only if thread is first waiter or empty
     */
    protected final boolean tryAcquire(int acquires) {
        if (getState() == 0 && !hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }
}
```

可以看到，与非公平锁不同的是，在公平锁在决定是否要去抢锁之前会有一个额外的判断，也就是会调用 `hasQueuedPredecessors()` 方法，这个方法的作用在于判断队列中是否有等待处理的线程，如果没有，当前线程才会去抢锁，如果队列中有线程则无法抢锁，这样就保证了，线程可以按照自己在队列中的顺序公平地获得锁。  

