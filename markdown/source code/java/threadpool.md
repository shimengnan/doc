# 线程池

线程池主要解决两个问题:

​	针对大量的异步任务,提高性能,减少每个任务 创建销毁调度线程的开销.

​	提供了一种限制和管理资源的方法.



![类结构图](..\..\doc\uml\threadpool\ScheduledThreadPoolExecutor.png)

## Executor 

提供execute 方法用来执行异步任务

```java
 /*
 * @since 1.5
 * @author Doug Lea
 */
public interface Executor {
    /**
    *异步执行command在将来
    *缺陷:不提供有返回值的任务
    */
    void execute(Runnable command);
}
```

## ExecutorService 

扩展了Executor 接口,增加了Future 返回对象

```java
 /
 * 扩展了Executor 的行为   
 * @since 1.5
 * @author Doug Lea
 */
public interface ExecutorService extends Executor {
	/**
	*启动已提交任务的有序关闭,不在接收新任务
	*/
	void shutdown();
	/**
	*尝试停止所有正在执行的任务,暂停在队列中等待执行的任务,并返回等待执行任务的集合
	*/
	List<Runnable> shutdownNow();
	/**
	*executor 是否 shut down
	*/
	boolean isShutdown();
	/**
	*是否所有的task shut down
	*/
	boolean isTerminated();
	/**
	*阻塞,直到所有的task 在收到shutdown 请求后执行结束或者超时
	*/
	boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
    /**
    *提交任务(Callable)-有返回值
    */
    <T> Future<T> submit(Callable<T> task);
    /**
    *提交任务-返回预期(参数)结果
    */
    <T> Future<T> submit(Runnable task, T result);
    /**
    *提交任务(Runnable)-Future 返回值null
    */
    Future<?> submit(Runnable task);
    /**
    *执行任务集合--全部任务执行结束返回
    */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
     /**
     *执行任务集合--全部任务执行结束返回或到达超时时间返回
     */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;
     /**
     *执行任务集合--当任务结合中执行成功一条即返回.其余任务取消执行
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;
    /**
     *执行任务集合--当任务结合中执行成功一条即返回或超时.其余任务取消执行
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```



## ThreadPoolExecutor

线程池的基本实现

```java
/**
* 标准线程池实现
* ThreadPoolExecutor 可以通过调整参数来扩展灵活性
* Executors.newCachedThreadPool 无上线线程池,自动回收线程
* Executors.newFixedThreadPool  固定大小线程池
* Executors.newSingleThreadExecutor 单个后台线程 
* ****按需创建****
* 默认来说,每个核心线程初始化通常在提交任务时.但也可以通过prestartCoreThread显示的初始化线程.
*/
public class ThreadPoolExecutor extends AbstractExecutorService {
    /**
     *高2位代表 runState
     *RUNNING 运行状态,接收所有任务  
     *SHUTDOWN 不接收新任务,处理队列中的任务
     *STOP 不接收新任务,不处理队列中任务,中断当前任务
     *TIDYING 所有任务terminated,workerCount 为0,线程过度到TIDYING 状态将通过调用 terminated方法
     *TERMINATED terminated()方法执行结束
     *低29位代表 线程数量--(线程已经创建,未销毁)
     *
     */
	private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    /**
     * 任务队列
     * BlockingQueue 用来存储提交的任务
	 * 如果线程池pool size小于corePool size,这时提交的任务会构建新线程,而不是存入队列;
	 * 如果线程池pool size大于corePool size,这时提交的任务会存入队列,而不是新插入线程;
	 * 如果任务不能存入队列(队列已满),如果线程池pool size没有达到maximum pool size,将会创建线程,否则执行reject 策略
	 * 如果提交的任务存入队列,如果pool size 没达到 maximum Pool size 将会创建一个新线程
     */
    private final BlockingQueue<Runnable> workQueue;
    /**
     * 线程工厂
     * 线程通过ThreadFactory 创建,如果没有特殊指定使用Executors.defaultThreadFactory
     */
    private volatile ThreadFactory threadFactory;
    /**
     * 核心参数---拒绝策略
     * 当线程池关闭,或队列已满,且线程池大小已达到maximumPoolSize 时
     * AbortPolicy   中断 --默认策略
     * CallerRunsPolicy 调用者线程执行该任务
     * DiscardPolicy	丢弃任务
     * DiscardOldestPolicy	丢弃workQueue头部任务
     */
    private volatile RejectedExecutionHandler handler;
    /**
     * 核心参数--线程池核心线程数
     * 当一个任务提交时,如果线程池大小小于 corePoolSize,这时会创建一个新线程用来处理这个任务.即便线程池中的线程是空闲的;
	 * 如果线程池大小大于corePoolSize 小于 maximumPoolSize,如果任务队列满了将会创建新的线程;通过将corePoolSize 与 maximumPoolSize 设置相同,可以创建一个固定大小的线程池;
	 * maximumPoolSize 设置成 Integer.MAX_VALUE 可以将代表是无界线程池,
	 * 通常,core 与 maximum pool size 在构造方法中赋值,但也可以通过setCorePoolSize 与 setMaximumPoolSize 来动态设置
     */
    private volatile int corePoolSize;

    /**
     * 核心参数--线程池最大线程数
     */
    private volatile int maximumPoolSize;
    /**
     * 核心参数--线程keepAliveTime 时间
     * 如果线程池中当前线程的数量大于corePoolSize 的数量,额外的线程将被销毁.通过keepAliveTime配置,单位通过TimeUnit配置.keepAliveTimes 只有当 maximum core pool size 不一致时生效.
     */
    private volatile long keepAliveTime;
    
}
```

submit 方法 源码逻辑如下

```java
step 1.<ExecutorService>
Future<?> submit(Runnable task);

step 2.<AbstractExecutorService>
public Future<?> submit(Runnable task) {
	if (task == null) throw new NullPointerException();
	//创建FutureTask
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}
step 3.<Executor>
void execute(Runnable command)
    
step 4.<ThreadPoolExecutor>
public void execute(Runnable command){
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
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) { //提交我们的额任务到workQueue
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false)) //使用maximumPoolSize作为边界
            reject(command); //还不行？拒绝提交的任务
    }
}
step 5.<ThreadPoolExecutor>
private boolean addWorker(Runnable firstTask,boolean core){
	
	w= new Worker(firstTask);
	final Thread t= w.thread;
	workers.add(w);
	t.start();
}
//上述代码是简化的.实际情况复杂很多
```

## FutureTask 

FutureTask 用于封装任务(Runnable,Callable 接口,统一返回值Future)

![](..\..\doc\uml\threadpool\FutureTask.png)





## Worker

```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable{
    /** worker 的工作线程*/
    final Thread thread;
    /** 需要运行的任务. */
    Runnable firstTask;
    public void run() {
    	runWorker(this);
    }
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
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
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
    /** 循环从workQueue 中获取任务执行**/
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
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {// 核心
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
}
```



## ShecduledExecutorService

扩展了ExecutorService 增加了调度功能-延迟执行,预期执行

```java
public interface ScheduledExecutorService extends ExecutorService {
    /**
    *在 delay* unit 时间延迟后执行command
    */
    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay, TimeUnit unit);
    /**
    *在 delay* unit 时间延迟后执行call
    */
    public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay, TimeUnit unit);
    /**
    * 经过initialDelay * unit 后
    * 每隔 period * unit 执行command
    * (假设,上一次的执行时间为 cost 如果 cost > period 那么下一次执行需要等待上一次结束)
    * 第一次 initialDelay + period  第二次 initialDelay + 2*period ...
    * 无论任务耗时多久不影响下一次的执行.
    */
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);
    /**
    * 经过initialDelay * unit 后
    * 第一次执行. 第一次执行成功后,经过delay
    * 第二次执行. 第二次执行成功后....
    */
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit);
}
```

## ScheduledThreadPoolExecutor

ScheduledThreadPoolExecutor 用于延时执行 或 周期执行任务调度

```java
public class ScheduledThreadPoolExecutor
        extends ThreadPoolExecutor
        implements ScheduledExecutorService {
    /**
    *构造方法
    */
    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       ThreadFactory threadFactory,
                                       RejectedExecutionHandler handler) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), threadFactory, handler);
    }    
    /**
     * 延迟delay 执行command
     */
    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay,
                                       TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
            
        RunnableScheduledFuture<?> t = decorateTask(command,
            new ScheduledFutureTask<Void>(command, null,
                                          triggerTime(delay, unit)));
        delayedExecute(t);
        return t;
    }

    /**
     * 延迟delay 执行callable
     */
    public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay,
                                           TimeUnit unit) {
        if (callable == null || unit == null)
            throw new NullPointerException();
        RunnableScheduledFuture<V> t = decorateTask(callable,
            new ScheduledFutureTask<V>(callable,
                                       triggerTime(delay, unit)));
        delayedExecute(t);
        return t;
    }

    /**
     * 使用固定频率
     */
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (period <= 0)
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(period));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }

    /**
     * 使用固定延迟调度
     */
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (delay <= 0)
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(-delay));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }
    /**
    * 上述方法最终都会调用方法:delayedExecute
    */
     private void delayedExecute(RunnableScheduledFuture<?> task) {
        if (isShutdown())
            reject(task);//拒绝任务
        else {
            super.getQueue().add(task);//添加任务-- 
            
            if (isShutdown() &&
                !canRunInCurrentRunState(task.isPeriodic()) &&
                remove(task))
                //任务已经添加队列,线程池已经关闭,取消并移除任务
                task.cancel(false);
            else
                ensurePrestart();//确保启动线程
        }
    }
    /**
    *确保启动线程
    */
    void ensurePrestart() {
        int wc = workerCountOf(ctl.get());
        if (wc < corePoolSize)
            addWorker(null, true);
        else if (wc == 0)
            addWorker(null, false);
    }
    //addWorker 后,Worker 最终会执行task.run 方法.
    /**
    *调度任务封装
    */
	private class ScheduledFutureTask<V>
            extends FutureTask<V> implements RunnableScheduledFuture<V> {
        /** 用于先进先出的序号 */
        private final long sequenceNumber;

        /** 下一次执行的毫秒数 */
        private long time;

        /**
         * fixed rate or delay 周期
         */
        private final long period;

        /** 用于 循环调用的 self 引用 */
        RunnableScheduledFuture<V> outerTask = this;	
        
        /**
         *用于在队列中排序,靠近调度时间的在前面
         */
        public int compareTo(Delayed other) {
            if (other == this) // compare zero if same object
                return 0;
            if (other instanceof ScheduledFutureTask) {
                ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
                long diff = time - x.time;
                if (diff < 0)
                    return -1;
                else if (diff > 0)
                    return 1;
                else if (sequenceNumber < x.sequenceNumber)
                    return -1;
                else
                    return 1;
            }
            long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
            return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
        }
        /**
         *调度任务run 方法,由 Worker 中的run 方法触发调用
         */
        public void run() {
        	//判断是否为周期执行方法
            boolean periodic = isPeriodic();
            if (!canRunInCurrentRunState(periodic))
            	//判断当前状态是否需要取消
                cancel(false);
            else if (!periodic)
            	//不是周期运行,仅执行一次
                ScheduledFutureTask.super.run();
            else if (ScheduledFutureTask.super.runAndReset()) {
				//是周期运行,执行并重置线程状态,并初始化下次调度时间
				//设置下一周期执行时间
                setNextRunTime();
                //循环调用--将自己的引用加入到调度队列中
                reExecutePeriodic(outerTask);
            }
        }
        /**
         *重新指定下一周期执行时间
         */
        void reExecutePeriodic(RunnableScheduledFuture<?> task) {
            if (canRunInCurrentRunState(true)) {
            	//将下一次调度任务添加到队列
                super.getQueue().add(task);
                if (!canRunInCurrentRunState(true) && remove(task))
                    task.cancel(false);
                else
                	//确保线程启动
                    ensurePrestart();
            }
    	}
    	/** 获取延迟时间 **/
    	public long getDelay(TimeUnit unit) {
            return unit.convert(time - now(), NANOSECONDS);
        }
	}
	/**
	 * 延迟队列
	 * 基于堆的数据结构
	 */
	static class DelayedWorkQueue extends AbstractQueue<Runnable>
        implements BlockingQueue<Runnable> {
        /**
        * 初始队列容量 初始化队列
        */
        private static final int INITIAL_CAPACITY = 16;
        private RunnableScheduledFuture<?>[] queue =
            new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
        private final ReentrantLock lock = new ReentrantLock();
        /**
         * Condition signalled when a newer task becomes available at the
         * head of the queue or a new thread may need to become leader.
         */
        private final Condition available = lock.newCondition();
        /**
        *获取任务
        */
        public RunnableScheduledFuture<?> take() throws InterruptedException {
            final ReentrantLock lock = this.lock;
            lock.lockInterruptibly();
            try {
                for (;;) {
                	//获取队首元素
                    RunnableScheduledFuture<?> first = queue[0];
                    if (first == null)
                    	//如果是空阻塞等待
                        available.await();
                    else {
                    	//不是空
                        long delay = first.getDelay(NANOSECONDS);
                        //已到调度时间-当前需要执行任务
                        if (delay <= 0)
                        	//***最终出口,返回需要执行的任务
                            return finishPoll(first);
                        first = null; // don't retain ref while waiting
                        if (leader != null)
                        	//未到调度时间
                            available.await();
                        else {
                            Thread thisThread = Thread.currentThread();
                            leader = thisThread;
                            try {
                            	//阻塞延迟时间
                                available.awaitNanos(delay);
                            } finally {
                                if (leader == thisThread)
                                    leader = null;
                            }
                        }
                    }
                }
            } finally {
                if (leader == null && queue[0] != null)
                    available.signal();
                lock.unlock();
            }
        }
        /**
        * 
        */
        private RunnableScheduledFuture<?> finishPoll(RunnableScheduledFuture<?> f) {
            int s = --size;
            RunnableScheduledFuture<?> x = queue[s];
            queue[s] = null;
            if (s != 0)
                siftDown(0, x);
            setIndex(f, -1);
            return f;
        }
        /**
         * 从k开始堆向上调整
         */
        private void siftUp(int k, RunnableScheduledFuture<?> key) {
            while (k > 0) {
                int parent = (k - 1) >>> 1;
                RunnableScheduledFuture<?> e = queue[parent];
                if (key.compareTo(e) >= 0)	
                    break;
                queue[k] = e;
                setIndex(e, k);
                k = parent;
            }
            queue[k] = key;
            setIndex(key, k);
        }

        /**
         * 从k堆向下调整
         */
        private void siftDown(int k, RunnableScheduledFuture<?> key) {
            int half = size >>> 1; 4
            while (k < half) {0<4
                int child = (k << 1) + 1;9
                RunnableScheduledFuture<?> c = queue[child];
                int right = child + 1;
                if (right < size && c.compareTo(queue[right]) > 0)
                    c = queue[child = right];
                if (key.compareTo(c) <= 0)
                    break;
                queue[k] = c;
                setIndex(c, k);
                k = child;
            }
            queue[k] = key;
            setIndex(key, k);
        }
    }
}        
```

## 关键词

线程池, 调度线程池,延迟队列,最小堆

## 参考

https://www.jianshu.com/p/925dba9f5969

https://www.jianshu.com/p/5d5198b434a2