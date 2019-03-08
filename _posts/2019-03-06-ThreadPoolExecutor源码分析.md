---
layout:     post
title:      ThreadPoolExecutor线程池源码分析
subtitle:   ThreadPoolExecutor
date:       2019-03-06
author:     BY
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - ThreadPool
---

## 前言

在并发很大的情况下，频繁创建线程与销毁线程会带来很大的消耗，线程池就是在解决这个问题的。


## 线程构造函数意义
```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
``` 

- corePoolSize:核心线程的数量
- maximumPoolSize:最大线程数
- keepAliveTime:闲置线程被回收的时间限制 
- unit:时间单位（ms、us什么的）
- workQueue:消息队列（存储没有线程的任务，存储的任务需等待其他任务完成释放的线程）
- threadFactory:线程工厂,用来给线程去个有意义的名字
- handler:拒绝策略，即当加入线程失败，采用该handler来处理

当我们看到corePoolSize与maximumPoolSize的时候，可能会感到疑惑，那么当前线程数poolSize与
corePoolSize和maximumPoolSize有什么关系呢？
当新提交一个任务时：
- 如果poolSize<corePoolSize，新增加一个线程处理新的任务。
- 如果poolSize=corePoolSize，新任务会被放入阻塞队列等待。
- 如果阻塞队列的容量达到上限，且这时poolSize<maximumPoolSize，新增线程来处理任务。
- 如果阻塞队列满了，且poolSize=maximumPoolSize，那么线程池已经达到极限，会根据饱和策略RejectedExecutionHandler拒绝新的任务。


## 线程池状态
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
上面的代码是在jdk11里的，线程的池的状态有上面的RUNNING、SHUTDOWN、STOP、TIDYING、TERMINATED
五种状态，他们都是用高三位在储存线程池的状态，以剩下的29位数来储存当前线程数，即最大线程数位2的29次方-1。

- RUNNING:接受新的任务并且处理队列的任务。
- SHUTDOWN:不接受任务，但处理已经进入队列的任务。
- STOP:不接受新任务，也不处理队列里的任务，并且中断正在执行的任务。
- TIDYING:所有任务执行完成。
- TERMINATED:terminated()已经执行完成。 

状态间的相互转换：
- RUNNING -> SHUTDOWN:可以调用 shutdown 方法
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
- RUNNING 或者 SHURDOWN->STOP:调用
```$xslt
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
- SHUTDOWN -> TIDYING：当队列和线程池为空 
- STOP -> TIDYING ：当线程池为空 
- TIDYING -> TERMINATED ：当terminated()钩子方法执行完成 
```$xslt
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
execute方法：
```
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         * 如果任务能添加进队列，我们仍然需要进行双重检查
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         * 如果我们没有任务，可以尝试添加一个线程，如果添加失败，使用拒绝策略
         */
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
            //双重检查，状态不为RUNNING,且移除任务成功
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
addWorker









                          
                          
                          
                    
