---

title: 啃透Java并发之线程池篇
date: 2019-01-06 17:13:43
categories: "Java并发编程"
tags: [Java,并发,多线程]
---

<Excerpt in index | 首页摘要> 

# 背景

​	Java中的线程池是运用场景最多的并发框架，几乎所有需要异步或并发执行任务的程序都可以使用线程池。

<!-- more -->
<The rest of contents | 余下全文>



# 使用线程池的好处

1. 降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
2. 提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。
3. 提高线程的可管理性。线程是稀缺资源，如果无限制地创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一分配、调优和监控。
4. 

# 线程池的实现原理

![](https://note.youdao.com/yws/api/personal/file/64BC22589F0A4C809EAD6D943A995183?method=download&shareKey=f2f9d5b8bbe5189d462e277b4027a339)

1. 线程池判断核心线程池里的线程是否都在执行任务。如果不是，则创建一个新的工作线程来执行任务。如果核心线程池里的线程都在执行任务，则进入下个流程。
2. 线程池判断工作队列是否已经满。如果工作队列没有满，则将新提交的任务存储在这个工作队列里。如果工作队列满了，则进入下个流程。
3. 线程池判断线程池的线程是否都处于工作状态。如果没有，则创建一个新的工作线程来执行任务。如果已经满了，则交给饱和策略来处理这个任务
4. 

# 线程池的主要成员及关系

​	主要有任务执行机制的核心接口Executor，以及继承自Executor的ExecutorService接口。Executor框架有两个关键类实现了ExecutorService接口（ThreadPoolExecutor和ScheduledThreadPoolExecutor）。

他们之间的关系图如下：

![](https://note.youdao.com/yws/api/personal/file/51F097CD8DCB4AAC81C15328B272E7A3?method=download&shareKey=7edc5da6635b8c13294105ce58901e9c)

## ThreadPoolExecutor

​	ThreadPoolExecutor通常使用工厂类Executors来创建。Executors可以创建3种类ThreadPoolExecutor：SingleThreadExecutor、FixedThreadPool和CachedThreadPool。

* FixedThreadPool。下面是Executors提供的，创建使用固定线程数的FixedThreadPool的API。FixedThreadPool适用于为了满足资源管理的需求，而需要限制当前线程数量的应用场景，它适用于负载比较重的服务器。
* SingleThreadExecutor。下面是Executors提供的，创建使用单个线程的SingleThread-Executor的API。SingleThreadExecutor适用于需要保证顺序地执行各个任务；并且在任意时间点，不会有多个线程是活动的应用场景。
* CachedThreadPool。下面是Executors提供的，创建一个会根据需要创建新线程的CachedThreadPool的API。CachedThreadPool是大小无界的线程池，适用于执行很多的短期异步任务的小程序，或者是负载较轻的服务器。

## ScheduledThreadPoolExecutor

​	ScheduledThreadPoolExecutor通常使用工厂类Executors来创建。Executors可以创建2种类型的ScheduledThreadPoolExecutor，适用于需要多个后台线程执行周期任务，同时为了满足资源管理的需求而需要限制后台线程的数量的应用场景。

* ScheduledThreadPoolExecutor。包含若干个线程的ScheduledThreadPoolExecutor。
* SingleThreadScheduledExecutor。只包含一个线程的ScheduledThreadPoolExecutor。
* 

# 线程池的使用及源码分析

## 线程池的5中状态

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));    
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

```

其中AtomicInteger变量ctl是用来记录线程池状态和数量的：利用低29位表示线程池中线程数，通过高3位表示线程池的运行状态

1. RUNNING：-1 << COUNT_BITS，即高3位为111，该状态的线程池会接收新任务，并处理阻塞队列中的任务；
2. SHUTDOWN： 0 << COUNT_BITS，即高3位为000，该状态的线程池不会接收新任务，但会处理阻塞队列中的任务；
3. STOP ： 1 << COUNT_BITS，即高3位为001，该状态的线程不会接收新任务，也不会处理阻塞队列中的任务，而且会中断正在运行的任务；
4. TIDYING ： 2 << COUNT_BITS，即高3位为010, 所有的任务都已经终止；
5. TERMINATED： 3 << COUNT_BITS，即高3位为011, terminated()方法已经执行完成；

## 状态之间的转换

![](https://note.youdao.com/yws/api/personal/file/70263B2765A346F996D41B19B7101882?method=download&shareKey=3f853a545f65daa46d61d29ba0fc8b2d)

## 线程池的创建

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

创建主要是一些相关参数的初始化，我们来看下这些参数的作用

1. corePoolSize（线程池的基本大小）：当提交一个任务到线程池时，线程池会创建一个线程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程，等到需要执行的任务数大于线程池基本大小时就不再创建。如果调用了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有基本线程。
2. maximumPoolSize（线程池最大数量）：线程池允许创建的最大线程数。如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。值得注意的是，如果使用了无界的任务队列这个参数就没什么效果。
3. keepAliveTime（线程活动保持时间）：线程池的工作线程空闲后，保持存活的时间。所以，如果任务很多，并且每个任务执行的时间比较短，可以调大时间，提高线程的利用率。
4. unit（线程活动保持时间的单位）：可选的单位有天（DAYS）、小时（HOURS）、分钟（MINUTES）、毫秒（MILLISECONDS）、微秒（MICROSECONDS，千分之一毫秒）和纳秒（NANOSECONDS，千分之一微秒）。
5. workQueue（任务队列）：用于保存等待执行的任务的阻塞队列。可以选择以下几个阻塞队列。
   * ArrayBlockingQueue：是一个基于数组结构的有界阻塞队列，此队列按FIFO（先进先出）原则对元素进行排序。
   * LinkedBlockingQueue：一个基于链表结构的阻塞队列，此队列按FIFO排序元素，吞吐量通常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队列。
   * SynchronousQueue：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于Linked-BlockingQueue，静态工厂方法Executors.newCachedThreadPool使用了这个队列。
   * PriorityBlockingQueue：一个具有优先级的无限阻塞队列。
6. threadFactory：用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设置更有意义的名字。
7. handler（饱和策略）：当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。这个策略默认情况下是AbortPolicy，表示无法处理新任务时抛出异常。在JDK 1.8中Java线程池框架提供了以下4种策略。
   * AbortPolicy：直接抛出异常。
   * CallerRunsPolicy：只用调用者所在线程来运行任务。
   * DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务。
   * DiscardPolicy：不处理，丢弃掉。

当然，也可以根据应用场景需要来实现RejectedExecutionHandler接口自定义策略。

## 线程池的执行

### execute方法

```java
 public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        //如果工作线程小于corePoolSize，则直接添加线程，添加成功后返回
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //到这里肯定是不能直接添加线程的，核心线程数已达corePoolSize，判断线程处于运行状态，                                      是否能添加任务至阻塞队列。
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            //如果线程池未处于运行状态，将任务从阻塞队列中移除，并执行拒绝策略
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //如果排队失败（有界的阻塞队列）,说明阻塞队列已满，则添加一个非核心态的worker
        //如果非核心线程数量达到maximumPoolSize，则执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
```

### 添加工作线程addWork方法

```java
 private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);
            //线程状态的一些判断，如果线程不是运行状态，返回false
            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
            /* 数量判断
             * 如果当前新增的是核心态的worker则与corePoolSize进行比较
             * 如果当期新增的是非核心态的worker则与maximumPoolSize进行比较
             * 不满足数量限制则直接添加失败，进入后续的排队或者执行拒绝策略
             */
            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
        //执行到这里，说明满足创建线程的条件，worker数量成功加一，然后进行线程的创建
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            //通过Worker内部类封装任务
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
                    //添加时再次进行线程池状态检查，线程状态检查
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    //执行任务，调用worker的run方法
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

从这两段源码，大概能够弄明白线程池执行的原理和流程，创建线程和执行线程。更加详细的内容各位同学可继续详细源码

下面在补充一下未能成功创建线程，但是成功添加至阻塞队列的任务是怎么获取的

### getTask

```java
private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?
 
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);
 
            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }
 
            int wc = workerCountOf(c);
 
            // Are workers subject to culling?
            //这里判断基本线程数量是否大于corePoolSize，大于corePoolSize，任务才会被添加到
            //阻塞队列
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
 
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }
 
            try {
                //主要是在这里从阻塞队列获取任务，
                //如果有设置超时时间，则调用poll方法获取，超时为空返回false
                //如果没有设置超时时间，则一直堵塞至有任务可取
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

## 线程池的关闭

* shutdown ：将线程池里的线程状态设置成SHUTDOWN状态, 然后中断所有没有正在执行任务的线程。
* shutdownNow ：将线程池里的线程状态设置成STOP状态, 然后停止所有正在执行或暂停任务的线程。

### shutdown

```java
public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //一些安全检查
            checkShutdownAccess();
            //设置线程池状态为SHUTDOWN
            advanceRunState(SHUTDOWN);
            //通过循环给所有work加上中断标识（t.interrupt()）
            //这里需要注意，通过interrupt方法来中断线程，能响应的就会中断，无法响应中断的任务
            //可能永远无法终止。
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }
```

### shuwdownNow

```java
public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            interruptWorkers();
            //和shutdown的主要区别在这里
            //shutdown会将阻塞队列中的任务清掉，之后不会继续执行
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
```

## 线程池的监控

​	如果在系统中大量使用线程池，则有必要对线程池进行监控，方便在出现问题时，可以根据线程池的使用状况快速定位问题。可以通过线程池提供的参数进行监控，在监控线程池的时候可以使用以下属性。

* taskCount：线程池需要执行的任务数量。
* completedTaskCount：线程池在运行过程中已完成的任务数量，小于或等于taskCount。
* largestPoolSize：线程池里曾经创建过的最大线程数量。通过这个数据可以知道线程池是否曾经满过。如该数值等于线程池的最大大小，则表示线程池曾经满过。
* getPoolSize：线程池的线程数量。如果线程池不销毁的话，线程池里的线程不会自动销毁，所以这个大小只增不减。
* getActiveCount：获取活动的线程数。

# 总结

​	通过阅读jdk源码，能够更好的理解线程池的实现原理和相关配置（主要是防止cpu资源耗尽或者利用率较低等。。）。也能够在实际中更准确，更有效的使用线程池



参考书籍《java并发编程的艺术》

参考博客：<https://www.jianshu.com/p/bf4a9e0b9e60>