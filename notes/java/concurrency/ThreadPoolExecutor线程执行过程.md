#  ThreadPoolExecutor线程执行过程

ThreadPoolExecutor构造器为：

```java
/**
 * corePoolSize：核心线程池大小
 * maximumPoolSize：最大线程池大小
 * keepAliveTime：线程池中超过corePoolSize数目的空闲线程最大存活时间
 * unit：keepAliveTime时间单位----TimeUnit.MINUTES
 * workQueue：阻塞队列----new ArrayBlockingQueue<Runnable>(20)====20容量的阻塞队列
 * threadFactory：新建线程工厂
 * rejectedExecutionHandler：当提交任务数超过maxmumPoolSize+workQueue之和时,即当提交第41个任务时执行RejectedExecutionHandler.rejectedExecution()方法对提交任务做处理。默认的RejectedExecutionHandler为 {@code AbortPolicy}
*/
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    ......
}
```

构造对象如下：

```java
ExecutorService es = new ThreadPoolExecutor(5,10,5,TimeUnit.SECONDS, new ArrayBlockingQueue(10));
//往es中添加21个任务；
for(21){
    es.execute(new Runnable() {...});
}
es.shutdown();
```

则线程池里最多可同时存在：`corePoolSize+(maximumPoolSize-corePoolSize) +workQueue.size()`= `5+(10-5)+10` = 20个任务。且其中`maximumPoolSize`(10)个任务处于运行中，其他的任务处于等待队列中。再具体点，第1至5和16至20号共十个任务处于运行中，6至15号共计10个任务处于等待队列。添加第21个任务时，只有当线程池总任务数小于20（即有任务执行完成）时才能添加成功，否则21号任务会被拒绝（默认`RejectedExecutionHandler`策略）。

原因如下：

> 1.当线程池小于corePoolSize时，新提交任务将创建一个新线程执行任务，即使此时线程池中存在空闲线程。 
> 2.当线程池达到corePoolSize时，新提交任务将被放入workQueue中，等待线程池中任务调度执行 
> 3.当workQueue已满，且maximumPoolSize>corePoolSize时，新提交任务会创建新线程执行任务 
> 4.当提交任务数超过maximumPoolSize时，新提交任务由RejectedExecutionHandler处理 
> 5.当线程池中超过corePoolSize线程，线程空闲时间达到keepAliveTime时，关闭空闲线程
> 6.当设置allowCoreThreadTimeOut(true)时，线程池中corePoolSize线程空闲时间达到keepAliveTime也将关闭 



下面看下线程具体执行过程

`execute()`方法

```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * 分3个步骤执行：
         *
         * 1.如果少于corePoolSize的线程正在运行，尝试
         * 启动一个新线程，使用给定Runnable实例作为线程
         * 第一个任务。对addWorker的调用会以原子方式检查runState和
         * workerCount，通过返回false来防止在不该增加新线程时增加
         * 线程的错误警告。
         *
         * 2.如果任务可以成功排队，那么我们仍然需要
         * 双重检查我们是否应该新建线程
         *（因为自上次检查后可能有线程死亡）或
         * 自进入此方法后线程池被关闭。 所以我们
         * 需要重新检查状态，如有必要，回滚入队任务，
         * 或者启动新线程。
         *
         * 3.如果无法将任务放入等待队列，那么尝试添加新的线程。 
         * 如果失败，那就说明线程池已关闭或饱和，任务会被拒绝。
         */
        int c = ctl.get();
    	//运行线程数小于corePoolSize，不排队
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
    	//尝试排队，注意：放入队列的是自定义Task而不是Worker包装！
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
    	//排队失败
        else if (!addWorker(command, false))
            reject(command);
    }
```



`addWorker()`方法

```java
/**
     * Checks if a new worker can be added with respect to current
     * pool state and the given bound (either core or maximum). If so,
     * the worker count is adjusted accordingly, and, if possible, a
     * new worker is created and started, running firstTask as its
     * first task. This method returns false if the pool is stopped or
     * eligible to shut down. It also returns false if the thread
     * factory fails to create a thread when asked.  If the thread
     * creation fails, either due to the thread factory returning
     * null, or due to an exception (typically OutOfMemoryError in
     * Thread.start()), we roll back cleanly.
     *
     * @param firstTask the task the new thread should run first (or
     * null if none). Workers are created with an initial first task
     * (in method execute()) to bypass queuing when there are fewer
     * than corePoolSize threads (in which case we always start one),
     * or when the queue is full (in which case we must bypass queue).
     * Initially idle threads are usually created via
     * prestartCoreThread or to replace other dying workers.
     *
     * @param core if true use corePoolSize as bound, else
     * maximumPoolSize. (A boolean indicator is used here rather than a
     * value to ensure reads of fresh values after checking other pool
     * state).
     * @return true if successful
     */
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

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

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            //这里要注意。
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
                    //重点，这个t是什么？
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



`Worker`类构造器：

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable{
        
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        //创建新线程，将当前worker对象放入该线程
        this.thread = getThreadFactory().newThread(this);
    }
    
    public void run() {
            runWorker(this);
        }
｝
```



那么，`addWorker()`方法里的`t.start()`就是运行Worker实例里的线程，该线程调用绑定的Worker实例的`run()`方法。而`run()`方法调用的是外部类`ThreadPoolExecutor`的`runWorker()` 方法。



再看`runWorker()`方法

```java
/**
 * Main worker run loop.  Repeatedly gets tasks from queue and
 * executes them, while coping with a number of issues:
 *
 * 1. We may start out with an initial task, in which case we
 * don't need to get the first one. Otherwise, as long as pool is
 * running, we get tasks from getTask. If it returns null then the
 * worker exits due to changed pool state or configuration
 * parameters.  Other exits result from exception throws in
 * external code, in which case completedAbruptly holds, which
 * usually leads processWorkerExit to replace this thread.
 *
 * 2. Before running any task, the lock is acquired to prevent
 * other pool interrupts while the task is executing, and then we
 * ensure that unless pool is stopping, this thread does not have
 * its interrupt set.
 *
 * 3. Each task run is preceded by a call to beforeExecute, which
 * might throw an exception, in which case we cause thread to die
 * (breaking loop with completedAbruptly true) without processing
 * the task.
 *
 * 4. Assuming beforeExecute completes normally, we run the task,
 * gathering any of its thrown exceptions to send to afterExecute.
 * We separately handle RuntimeException, Error (both of which the
 * specs guarantee that we trap) and arbitrary Throwables.
 * Because we cannot rethrow Throwables within Runnable.run, we
 * wrap them within Errors on the way out (to the thread's
 * UncaughtExceptionHandler).  Any thrown exception also
 * conservatively causes thread to die.
 *
 * 5. After task.run completes, we call afterExecute, which may
 * also throw an exception, which will also cause thread to
 * die. According to JLS Sec 14.20, this exception is the one that
 * will be in effect even if task.run throws.
 *
 * The net effect of the exception mechanics is that afterExecute
 * and the thread's UncaughtExceptionHandler have as accurate
 * information as we can provide about any problems encountered by
 * user code.
 *
 * @param w the worker
 */
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        //注意这个while条件，线程第一次运行时task为worker里绑定的我们提交的任务，
        //任务执行完以后就是用getTask()方法从workQueue里获取了
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
                    //直接同步调用，实现任务切换而工作线程不停止。因为这段代码是在工作线程里执行
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
                //置空，让while条件继续执行
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
```



`ThreadPoolExecutor`的`submit()`方法和`invokeAll()`方法不影响线程执行逻辑，不在解释。

**注意`submit()`方法会把`Task`用`FutureTask`包裹，而返回值又是`Future`。所以如果Task在工作线程里抛了异常，该异常会被`Future`持有，直到调用`Future.gt()`时才抛出。因此，调用`submit()`后必须遍历结果集，或者，不需要返值的`Task`不要用`submit()`执行，防止异常被吞。**



参考资料：

 [ThreadPoolExecutor使用详解](https://www.cnblogs.com/zedosu/p/6665306.html)