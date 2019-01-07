---
title: 啃透Java并发AQS源码分析篇
date: 2019-01-06 17:13:43
categories: "Java并发编程"
tags: [Java,并发,多线程]
---

<Excerpt in index | 首页摘要> 

# 概念
AQS:**队列同步器AbstractQueuedSynchronizer（以下简称同步器）**，是用来构建锁或者其他同步组件的基础框架，许多同步器可以通过AQS很容易的并且高效的构建出来。不仅RenntrantLock和Semaphore是基于AQS构建的，还包括CountDownLatch、ReentrantReadWriteLock、SynchronousQueue和FutureTask。

<!-- more -->
<The rest of contents | 余下全文>

同步器的主要使用方式是继承，子类通过继承同步器并实现它的抽象方法来管理同步状态，在抽象方法的实现过程中免不了要对同步状态进行更改，这时就需要使用同步器提供的3个方法（**getState()、setState(int newState)和compareAndSetState(int expect,int update)**）来进行操作，因为它们能够保证状态的改变是安全的。子类推荐被定义为自定义同步组件的静态内部类，同步器自身没有实现任何同步接口，它仅仅是定义了若干同步状态获取和释放的方法来供自定义同步组件使用，同步器既可以支持独占式地获取同步状态，也可以支持共享式地获取同步状态，这样就可以方便实现不同类型的同步组件（ReentrantLock、ReentrantReadWriteLock和CountDownLatch等）。

# 队列同步器的接口与实例
**重写同步器指定的方法时，需要使用同步器提供的如下3个方法来访问或修改同步状态。**
- getState()：获取当前同步状态。
- setState(int newState)：设置当前同步状态。
- compareAndSetState(int expect,int update)：使用CAS设置当前状态，该方法能够保证状态设置的原子性。

**同步器可重写的方法**
- protected boolean tryAcquire(int arg):独占式获取同步状态，实现该方法需要查询当前同步状态并判断同步状态是否符合预期，然后在进行CAS设置同步状态
- protected boolean tryRelease(int arg):独占式释放同步状态，等待获取同步状态的线程将同步状态有机会获取同步状态
- protected int tryAcquireShared(int arg):共享式获取同步状态，返回大于0的值，表示获取成功，反之，获取失败。
- protected boolean tryReleaseShared(int arg):共享式释放同步状态
- protected boolean isHeldExclusively():当前同步器是否在独占模式下被线程占用，一般该方法表示是否被当前线程所独占。

**同步器提供的模板方法**

| 方法名称                                          | 描述                                                         |
| ------------------------------------------------- | ------------------------------------------------------------ |
| void acquire(int arg)                             | 独占式获取同步状态，如果当前线程获取同步状态成功，则由该方法返回，否则，将进入同步队列等待，该方法将会调用重写的tryAcquire(int arg) |
| void acquireInterruptibly(int arg)                | 与acquire(int arg)相同，但是该方法响应中断，当前线程未获取到同步状态而进入同步队列中，如果当前线程被中断，则该方法会抛出InterruptException并返回 |
| boolean tryAcquireNanos(int arg,long nanos)       | 在acquireInterruptibly的基础上增加了超时限制，如果当前线程在超时时间内没有获取到同步状态，那么将会返回false,如果获取到了返回true |
| void acquireShared(int arg)                       | 共享式的获取同步状态。如果当前线程未获取到同步状态，将会进入同步队列等待，与独占式获取的主要区别是在同一时刻可以有多个线程获取到同步状态 |
| void acquireSharedInterruptibly(int arg)          | 与acquireShared(int arg)一样，该方法响应中断                 |
| boolean tryAcquireSharedNanos(int arg,long nanos) | 在acquireSharedInterruptibly的基础上增加了超时限制           |
| boolean release(int arg)                          | 独占式的释放同步状态，该方法会在释放同步状态之后，将同步队列中第一个节点包含的线程唤醒 |
| boolean releaseShared(int arg)                    | 共享式的释放同步状态                                         |
| Collecion\<Thread\> getQueuedThreads()            | 获取等待在同步队列上的线程集合                               |

同步器提供的模板方法基本上分为3类：**独占式获取与释放同步状态、共享式获取与释放同步状态和查询同步队列中的等待线程情况**。自定义同步组件将使用同步器提供的模板方法来实现自己的同步语义。

下面我们来看一个独占锁实现的例子

```java
public class Mutex implements Lock {
    // 静态内部类，自定义同步器
    private static class Sync extends AbstractQueuedSynchronizer {
        // 是否处于占用状态
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }
        // 当状态为0的时候获取锁
        public boolean tryAcquire(int acquires) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        // 释放锁，将状态设置为0
        protected boolean tryRelease(int releases) {
            if (getState() == 0) throw new
                    IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
        // 返回一个Condition，每个condition都包含了一个condition队列
        Condition newCondition() { return new ConditionObject(); }
    }
    // 仅需要将操作代理到Sync上即可
    private final Sync sync = new Sync();
    public void lock() { sync.acquire(1); }
    public boolean tryLock() { return sync.tryAcquire(1); }
    public void unlock() { sync.release(1); }
    public Condition newCondition() { return sync.newCondition(); }
    public boolean isLocked() { return sync.isHeldExclusively(); }
    public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
    public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
}

```
由于是实现独占锁，所以我们只需重写独占相关的方法（isHeldExclusively、tryAcquire、tryRelease），独占锁主要需要维护当前的status，基于AQS的模板方法，使得我们方便的实现独占锁。

我们可以稍微来看下AQS相关实现源码

```java
 public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            //死循环获取同步状态
            for (;;) {
                final Node p = node.predecessor();
                //当节点为头节点时，才尝试获取同步状态
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //非头节点线程进入等待头结点执行完成后唤醒
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

```java
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            //将当前节点CAS的方式设置为尾节点
            if (compareAndSetTail(pred, node)) {
                //将之前的尾节点的后继节点设置为当前节点
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

```java
private Node enq(final Node node) {
        //将当前节点设置为尾节点
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

```java
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                //唤醒等待的节点
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
​	上面的代码首先调用我们重写的tryAcquire获取同步状态，获取失败则构造同步节点（Node.EXCLUSIVE代表节点为独占类型），并通过addWaiter(Node node)方法将该节点加入到同步队列的尾部，最后调用acquireQueued(Node node,int arg)方法，使得该节点以“死循环”的方式获取同步状态。如果获取不到则阻塞节点中的线程，而被阻塞线程的唤醒主要依靠前驱节点的出队或阻塞线程被中断来实现（这里补充说明下，同步队列中通过Node表示当前线程，Node有前驱节点，后继节点）。

通过上面一些列代码，应该不难看出独占锁的实现原理，**主要还是通过AQS的状态属性来判断是否加锁以及释放锁，这里稍微对其他的同步组件利用state的方式简单说明**

​	**AQS负责管理同步器类中的状态**，他管理了一个整数状态信息，可以通过**getState**，**setState**，以及**compareAndSetState**等方法进行操作，这个整数可以用于表示任意状态。例如，ReentrantLock用它来表示所有的者线程已经重复获取该锁的次数，Semaphore用它来表示剩余的许可数量，FutureTask用它来表示任务的状态（尚未开始、正在运行、已完成、以及已取消）。在同步器类中还可以自行管理一些额外的状态变量，例如，ReentrantLock保存了当前所有者（当前线程）的信息，这样就能区分某个获取操作是重入还是竞争。

下面用共享的方式理解下上面话语的含义


```
public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```


```
 private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    //这里区别于独占锁对state的利用方式
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
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

# 总结
从上面源码及例子可以看出，其他各种基于AQS的同步组件，主要对state状态利用的方式不同，基础模板方式都是利用AQS自己的实现。通过AQS的源码实现，应该就能很好的理解JUC工具包下面的一些工具类的实现原理了，这里不在继续研究了，有兴趣的朋友可以自行查看源码（RenntrantLock、Semaphore、CountDownLatch等）

参考书籍
《java并发编程的艺术》
《java并发编程实战》