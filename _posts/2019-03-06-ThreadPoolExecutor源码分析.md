---
layout:     post
title:      ThreadPoolExecutor线程池源码分析
subtitle:   ThreadPoolExecutor
date:       2019-03-06
author:     BY
header-img: img/threadpool.jpeg
catalog: true
tags:
    - Thread
---

## 前言

在并发很大的情况下，频繁创建线程与销毁线程会带来很大的消耗，线程池就是在解决这个问题的。


## 线程构造函数

```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
``` 
#### 各参数的意义

- corePoolSize :核心线程的数量
- maximumPoolSize :最大线程数
- keepAliveTime :闲置线程被回收的时间限制 
- unit :时间单位（ms、us什么的）
- workQueue :消息队列（存储没有线程的任务，存储的任务需等待其他任务完成释放的线程）
- threadFactory :线程工厂,用来给线程取个有意义的名字
- handler :拒绝策略，即当加入线程失败，采用该 handler 来处理

#### corePoolSize 与 maximumPoolSize 

当我们看到 corePoolSize 与 maximumPoolSize 的时候，可能会感到疑惑，那么当前线程数 poolSize 与
corePoolSize 和 maximumPoolSize 有什么关系呢？

当新提交一个任务时：
- 如果 poolSize < corePoolSize，新增加一个线程处理新的任务。
- 如果 poolSize = corePoolSize，新任务会被放入阻塞队列等待。
- 如果阻塞队列的容量达到上限，且这时 poolSize < maximumPoolSize，新增线程来处理任务。
- 如果阻塞队列满了，且 poolSize = maximumPoolSize，那么线程池已经达到极限，会根据饱和策略 RejectedExecutionHandler 拒绝新的任务。


## 线程池状态

#### 常量与状态的源码

```
//高3位保存线程池状态，低29位保存线程数
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
//位移29位，即29 = Integer.Size - 3
private static final int COUNT_BITS = Integer.SIZE - 3;
//0001 1111 1111 1111 1111 1111 1111 1111
private static final int COUNT_MASK = (1 << COUNT_BITS) - 1;
// runState is stored in the high-order bits
//111 后29位全为0
private static final int RUNNING    = -1 << COUNT_BITS;
//000 后29位全为0
private static final int SHUTDOWN   =  0 << COUNT_BITS;
//001 后29位全为0
private static final int STOP       =  1 << COUNT_BITS;
//010 后29位全为0
private static final int TIDYING    =  2 << COUNT_BITS;
//011 后29位全为0
private static final int TERMINATED =  3 << COUNT_BITS;

//c：开头三位为000，后29位为线程数。~COUNT_MASK：111 后29位为0。
//c & ~COUNT_MASK得一个高3位为状态，低29位线程数的值，主要是根据c获取线程池状态
private static int runStateOf(int c)     { return c & ~COUNT_MASK; }
//c:同上。
private static int workerCountOf(int c)  { return c & COUNT_MASK; }
//一般用于设置状态并转移线程数，用于advanceRunState方法中。
private static int ctlOf(int rs, int wc) { return rs | wc; }
```
#### 线程池的5个状态

上面的代码是在jdk11里的，线程的池的状态有 `RUNNING`、 `SHUTDOWN`、 `STOP`、 `TIDYING`、 `TERMINATED`
这五种状态，他们都是使用int的高3位来储存线程池的状态，以低29位数来储存线程数，即最大线程数位2的29次方-1。
- RUNNING :接受新的任务并且处理队列的任务。
- SHUTDOWN :不接受任务，但处理已经进入队列的任务。
- STOP :不接受新任务，也不处理队列里的任务，并且中断正在执行的任务。
- TIDYING :所有任务执行完成。
- TERMINATED : terminated() 已经执行完成。 

#### 状态间的相互转换

- `RUNNING` -> `SHUTDOWN`:可以调用 shutdown 方法
- `RUNNING` 或者 `SHURDOWN` -> STOP:调用 shutdownNow
- `SHUTDOWN` -> `TIDYING`：当队列和线程池为空 
- `STOP` -> `TIDYING` ：当线程池为空 
- `TIDYING` -> `TERMINATED` ：当terminated()钩子方法执行完成 

```
public void shutdown() {
        //上锁
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            //设置线程状态，即ctl
            advanceRunState(SHUTDOWN);
            //给所有工作线程设置中断
            interruptIdleWorkers();
            //空方法
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        //尝试终止
        tryTerminate();
    }
}
```

```
public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            //设置线程状态，即ctl
            advanceRunState(STOP);
            //给所有工作线程设置中断
            interruptWorkers();
            //移除队列里的所有任务,tasks装的是所有未完成的任务
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        //尝试终止
        tryTerminate();
        return tasks;
    }

```

```
final void tryTerminate() {
        for (;;) {
            //获取当前线程池，线程数
            int c = ctl.get();
            //检查状态，c<RUNNING || c>=TIDYING || (c>=STOP && 队列不为空) 不进行处理
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateLessThan(c, STOP) && ! workQueue.isEmpty()))
                return;
              
            //当前线程数不为零
            if (workerCountOf(c) != 0) { // Eligible to terminate
                //只中断一次worker线程
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                //如果上面取得的线程数c与当前线程数ctl相等，将ctl设置为TIDYING
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        terminated();
                    } finally {
                        //terminated()方法完成，设置状态为TERMINATED
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```

## execute源码分析

execute()方法主要分为以下四种情况: 
- 情况1: 如果线程池内的有效线程数少于核心线程数 corePoolSize, 那么就创建并启动一个线程来
        执行新提交的任务. 
- 情况2: 如果线程池内的有效线程数达到了核心线程数 corePoolSize,并且线程池内的
        阻塞队列未满, 那么就将新提交的任务加入到该阻塞队列中. 
- 情况3: 如果线程池内的有效线程数达到了核心线程数 corePoolSize 但却小于最大线
        程数 maximumPoolSize, 并且线程池内的阻塞队列已满, 那么就创建并启动
        一个线程来执行新提交的任务. 
- 情况4: 如果线程池内的有效线程数达到了最大线程数 maximumPoolSize, 并且线程
        池内的阻塞队列已满, 那么就让 RejectedExecutionHandler 根据它的拒
        绝策略来处理该任务, 默认的处理方式是直接抛异常.

#### execute源码

```
public void execute(Runnable command) {
    int c = ctl.get();
    //当前线程数与核心线程比较，小于核心线程
    if (workerCountOf(c) < corePoolSize) {
        //添加工作线程    
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    //判断线程池状态是否为RUNNING与向队列添加任务
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        //双重检查，如果状态不为RUNNING且移除任务成功
        if (! isRunning(recheck) && remove(command))
            //使用拒绝策略
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        //添加工作线程失败，使用拒绝策略
        reject(command);
}

```

#### addWorker源码分析

```
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (int c = ctl.get();;) {
            // Check if queue empty only if necessary.
            /**
             * 1.状态为STOP以上的
             * 2.状态为SHUTDOWN，任务不为null的
             * 3.状态为SHUTDOWN，队列为空的
             */            
            if (runStateAtLeast(c, SHUTDOWN)
                && (runStateAtLeast(c, STOP)
                    || firstTask != null
                    || workQueue.isEmpty()))
                return false;
            
            for (;;) {
                //core为true,即当前线程数比较corePoolSize，用于前面的情况1,2
                //core为false,即当前线程数比较maximumPoolSize，用于前面的情况3,4
                if (workerCountOf(c)
                    >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
                    return false;
                //ctl加1,不成功一直循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateAtLeast(c, SHUTDOWN))
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
        //线程开启状态
        boolean workerStarted = false;
        //任务添加状态
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                //加锁
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int c = ctl.get();
                    //判断是否运行状态，或者    
                    if (isRunning(c) ||
                        (runStateLessThan(c, STOP) && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        //将work加入HashSet里
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                //启动线程
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            //添加线程失败，ctl减1，移除
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

## 线程消费队列里任务

`ThreadPoolExecutor`里`Worker`是线程的承载者，它继承与`AQS`,实现了`Runnable`的 run（）方法.

```
    public void run() {
        runWorker(this);
    }
```

#### runWorker源码分析
```
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        //获取任务
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            //循环从队列获取任务
            // 由前边可知, task 就是 w.firstTask
            // 如果 task为 null, 那么就不进入该 while循环, 也就不运行该 task. 如果    
            // task不为 null, 那么就执行 getTask()方法. 而getTask()方法是个无限
            // 循环, 会从阻塞队列 workQueue中不断取出任务来执行. 当阻塞队列 workQueue
            // 中所有的任务都被取完之后, 就结束下面的while循环.
            while (task != null || (task = getTask()) != null) {
                //锁住该线程
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    try {
                        //执行任务
                        task.run();
                        afterExecute(task, null);
                    } catch (Throwable ex) {
                        afterExecute(task, ex);
                        throw ex;
                    }
                } finally {
                    // 将 task 置为 null, 这样使得 while循环是否继续执行的判断, 就只能依赖于判断
                    // 第二个条件, 也就是 (task = getTask()) != null 这个条件, 是否满足.
                    task = null;
                    //任务运行完成后，在完成任务数上+1
                    w.completedTasks++;
                    //释放锁
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```
#### getTask源码分析
```
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            //获取线程状态与线程数
            int c = ctl.get();

            // Check if queue empty only if necessary.
            //1.线程状态为STOP以上，这时不处理队列里的任务
            //2.线程状态为SHUTDOWN，且队列为空，无任务可处理
            //上面两种情况，直接线程
            if (runStateAtLeast(c, SHUTDOWN)
                && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
                //线程数减一
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
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



## 拒绝策略

- `CallerRunsPolicy`：直接在本线程运行
- `AbortPolicy`：直接抛异常
- `DiscardPolicy`：不处理
- `DiscardOldestPolicy`：去除队列中的一个，执行 `execute` 方法
- 其他：可以实现 `RejectedExecutionHandler` 接口，自定义拒绝策略

## 不同线程池的优劣

Executors类则扮演着线程池工厂的角色,通过 Executors可以取得一个拥特定功能的线程池。


- newFixedThreadPool:：创建一个指定工作线程数量的线程池。每当提交一个任务就创建一个工作线程，如果工作线程
数量达到线程池初始的最大数，则将提交的任务存入到池队列中。
- newCachedThreadPool：创建一个可缓存的线程池。可创建的线程池基本没有限制，但如果线程空闲超过一定的时间，该
线程会自动终止。
- newSingleThreadExecutor：创建一个单线程化的Executor，即只创建唯一的工作者线程来执行任务，如果这个线程异
常结束，会有另一个取代它，保证顺序执行(我觉得这点是它的特色)。单工作线程最大的特点是可保证顺序地执行各个任务，
并且在任意给定的时间不会有多个线程是活动的 。
- newScheduleThreadPool：newScheduleThreadPool创建一个定长的线程池，而且支持定时的以及周期性的任务执行，类似于Timer。
                          
```
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```
```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new Synchrono　usQueue<Runnable>());
}
```
newSingleThreadExecutor 与 newScheduleThreadPool 分别被 FinalizableDelegatedExecutorService 与
ScheduledThreadPoolExecutor 包装。


## 关于线程池状态的思考

- 2019/03/06：为什么不把状态与线程数分开？为什么不用byte或者String来存储状态，用int来存储？
- 2019/03/09:addWorker的if (isRunning(c) ||(runStateLessThan(c, STOP) && firstTask == null))
为什么允许SHOTDOWN状态时添加任务？

## 参考
链接 ： [https://blog.csdn.net/cleverGump/article/details/50688008]()



                          
                          
                          
                    
