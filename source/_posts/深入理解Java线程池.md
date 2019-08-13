---
title: 深入理解Java线程池
date: 2019-07-29 15:22:23
tags:
categories: J.U.C
description: 对线程池的实现原理进行分析，并简单实现一个SimpleThreadPool
---

## 线程池介绍

**线程池存在的意义是什么？**
如果并发的线程数量很多，每个线程执行很短的时间就结束了，频繁的创建和销毁线程将会带来很大的开销。线程池实现了对**线程的复用**，每次线程执行任务完毕可以继续执行其他任务。

**什么时候使用线程池？**

- 单个任务时间短
- 需要处理的任务多

## ThreadPoolExecutor 类

J.U.C 并发包下的 ThreadPoolExecutor 是线程池的一个核心类，学习线程池有必要学习该类的源码。

![QQ20170331-004227.png](http://www.ideabuffer.cn/2017/04/04/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Java%E7%BA%BF%E7%A8%8B%E6%B1%A0%EF%BC%9AThreadPoolExecutor/QQ20170331-004227.png)

上面这张类图可清楚的看出 ThreadPoolExecutor、AbstractExecutorService、ExecutorService和Executor之间
的关系。其中 Executor 是一个顶层接口，只声明了一个 execute(Runnable) 方法，此方法用来执行传进去的任务。

ThreadPoolExecutor 的构造方法：

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        .....
    }
```

- **corePoolSize**: 核心线程数，当提交的任务超过核心线程数时则送入阻塞队列等待

- **maximumPoolSize**:  可开启的最大线程数；

- **workQueue**: 任务等待队列;

  ```java
  ArrayBlockingQueue;
  LinkedBlockingQueue;//一般使用这种方式，此时 maximumPoolSize 就不会起作用了
  SynchronousQueue;
  ```

- **keepAliveTime**: 当线程池中的线程数大于corePoolSize 时，如果线程空闲，核心线程外的线程超过时间后会自动销毁。

- **threadFactory**: 线程工厂。默认使用Executors.defaultThreadFactory。使用默认线程工厂时，线程具有相同的优先级并且为非守护线程，同时设置了线程名称。

- **handler**: **线程池饱和策略**。如果阻塞队列满了，并且线程池线程数 == maximumPoolSize ，这时如果有任务提交，则需要采取一种处理策略，线程池提供了 4 种策略：

  - AbortPolicy：直接抛出异常，这是默认策略；
  - CallerRunsPolicy：用调用者所在的线程来执行任务；
  - DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并执行当前任务;
  - DiscardPolicy：直接丢弃任务；

## 深入理解线程池实现原理

### 线程池状态

线程池共有五种状态：

1. **RUNNING**: 能接受新任务提交，并且能处理阻塞队列中的任务。
2. **SHUTDOWN**: 不能接受新提交任务，但是可以处理阻塞队列的任务，调用 shutdown() 进入。
3. **STOP**: 不能接受任务也不能处理阻塞队列中的任务，并且尝试终止当前任务，调用 shutdownNow() 进入。
4. **TIDYING**: 所有任务结束，则进入该状态。
5. **TERMINATED**： 结束状态。

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```

线程池状态转换图如下所示：

![threadpool-status.png](http://www.ideabuffer.cn/2017/04/04/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Java%E7%BA%BF%E7%A8%8B%E6%B1%A0%EF%BC%9AThreadPoolExecutor/threadpool-status.png)

### 任务执行

**execute()** 方法用来提交任务，源码分析如下：

```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
    	// ctl 记录着runState 和 workerCount
        int c = ctl.get();
    	// 如果 workerCount 小于 corePoolSize 则创建新的线程并添加该任务
        if (workerCountOf(c) < corePoolSize) {
            // addWorker 的第二个参数表示用 corePoolSize(true) 判断还是 maximumPoolSize(false) 判断
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
    	//如果线程池处于运行态，且成功添加到队列
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 添加进队列后，当线程池不处于运行态(被其他线程杀掉了)或此时线程池无线程，需要进行后处理
            // 不处于运行态则将任务移除队列
            // 线程池中无线程则添加一个空任务线程，来保证后续有线程执行该任务
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
    	// 执行到这有两种情况：
    	// 1. 线程池不是 RUNNING 状态，但 workerCount >= corePoolSize 并且 workerueue 已满
        else if (!addWorker(command, false))
            reject(command);
    }
```

execute执行流程如下：

![executor.png](http://www.ideabuffer.cn/2017/04/04/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Java%E7%BA%BF%E7%A8%8B%E6%B1%A0%EF%BC%9AThreadPoolExecutor/executor.png)

### 几种线程池创建方法

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

**newFixedThreadPool：**corePoolSize = nThread && maximumPoolSize = nThread，线程池维护 nThread 个线程。
**newSingleThreadExecutor：** corePoolSize = 1 && maximumPoolSize = 1，线程池只维护一个线程。
这两种创建方式缓冲队列都使用 LinkedBlockingQueue ，maximumPoolSize 不起作用。**newCachedThreadPool：**corePoolSize = 0 && maximumPoolSize = Integer.MAX_VALUE 使用 SynchronousQueue 没有容量，每次来了任务就创建线程运行。
**LinkedBlockingQueue 和 ArraBlockingQueue 区别 ：** 

1. 队列中锁的实现不同
   ArrayBlockingQueue中的锁是没有分离的，即生产和消费用的是同一个锁；
    LinkedBlockingQueue中的锁是分离的，即生产用的是putLock，消费是takeLock；
2. 在生产或消费时操作不同
   ArrayBlockingQueue 基于数组，在生产和消费的时候，是直接将枚举对象插入或移除的，不会产生或销毁任何额外的对象实例；
   LinkedBlockingQueue 基于链表，在生产和消费的时候，需要把枚举对象转换为 Node 进行插入或移除，会生成一个额外的Node对象，这在长时间内需要高效并发地处理大批量数据的系统中，其对于GC的存在影响；
3. 队列大小初始化不同
   ArrayBlockingQueue 是有界的,LinkedBlockingQueue 是无界的，默认是 Integer.MAX_VALUE；

### 合理配置线程池大小

**CPU 密集：**
CPU密集型只有多核裁员意义，应尽量配置可能小的线程和 CPU 个数相当，**CPU 个数 + 1**。
**IO 密集：**
IO 密集型存在很多IO阻塞，多于 CPU 数量的线程数能充分利用 CPU，**CPU 个数 * 2**

### 实现一个简单线程池

https://github.com/kilaOoO/JucPractice/tree/master/SimpleThreadPool