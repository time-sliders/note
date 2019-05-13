## sourcecode



## javadoc

An `ExecutorService` that executes each submitted task using one of possibly several pooled threads, normally configured using Executors factory methods.

Thread pools address`处理` two different problems: they usually provide improved performance when executing large numbers of asynchronous tasks, due to reduced per-task invocation`调用` overhead, and they provide a means of bounding`界定` and managing the resources, including threads, consumed when executing a collection of tasks. Each `ThreadPoolExecutor` also maintains some basic statistics`统计`, such as the number of completed tasks.

To be useful across a wide range of contexts, this class provides many **adjustable parameters** and **extensibility hooks**. However, programmers are urged to`推荐` use the more convenient Executors factory methods `Executors.newCachedThreadPool` (unbounded thread pool, with automatic thread reclamation), `Executors.newFixedThreadPool` (fixed size thread pool) and `Executors.newSingleThreadExecutor` (single background thread), that preconfigure settings for the most common usage scenarios. 

一般情况下不建议使用 Executors 提供的便利方法，出于以下原因考虑

1. **unbounded BlockingQueue容易造成OOM**
2. **推荐自定义 ThreadFactory 指定业务线程的 threadName**
3. **线程数不可控**

Otherwise, use the following guide when manually configuring and tuning this class:

**Core and maximum pool sizes**
A `ThreadPoolExecutor` will automatically adjust the pool size (see `getPoolSize`) according to the bounds set by **corePoolSize** (see `getCorePoolSize`) and **maximumPoolSize** (see `getMaximumPoolSize`). <u>When a new task is submitted in method `execute(Runnable)`, and fewer than **corePoolSize** threads are running, a new thread is created to handle the request, even if other worker threads are idle. If there are more than **corePoolSize** but less than **maximumPoolSize** threads running, a new thread will be created only if the queue is full. By setting **corePoolSize** and **maximumPoolSize** the same, you create a fixed-size thread pool. By setting maximumPoolSize to an essentially unbounded value such as `Integer.MAX_VALUE`, you allow the pool to accommodate`容纳` an arbitrary number of concurrent tasks. Most typically, core and maximum pool sizes are set only upon construction, but they may also be changed dynamically using `setCorePoolSize` and `setMaximumPoolSize`.</u>

**On-demand`按需` construction**
By default, even core threads are initially created and started only when new tasks arrive, but this can be overridden dynamically using method `prestartCoreThread` or `prestartAllCoreThreads`. You probably want to prestart threads if you construct the pool with a non-empty queue.

**Creating new threads**
New threads are created using a `ThreadFactory`. If not otherwise specified, a `Executors.defaultThreadFactory` is used, that creates threads to all be in the same `ThreadGroup` and with the same `NORM_PRIORITY` priority and `non-daemon` status. By supplying a different `ThreadFactory`, <u>you can alter the thread's name, thread group, priority, daemon status</u>, etc. If a `ThreadFactory` fails to create a thread when asked by returning `null` from newThread, the executor will continue, but might not be able to execute any tasks. **Threads should possess the "modifyThread" RuntimePermission**`具有…权限`. If worker threads or other threads using the pool do not possess this permission`如果使用池的工作线程或其他线程不具有此权限`, service may be degraded`降级`: configuration changes may not take effect in a timely manner`不能及时生效`, and a shutdown pool may remain in a state in which termination is possible but not completed.

**Keep-alive times**保持活跃时间
If the pool currently has more than **corePoolSize** threads, excess`多余的` threads will be terminated if they have been idle for more than the **keepAliveTime** (see `getKeepAliveTime(TimeUnit)`). This provides a means of reducing resource consumption`消耗` when the pool is not being actively used. If the pool becomes more active later, new threads will be constructed. This parameter can also be changed dynamically using method `setKeepAliveTime(long, TimeUnit)`. Using a value of `Long.MAX_VALUE` `TimeUnit.NANOSECONDS` effectively disables idle threads from ever terminating prior to shut down. By default, **the keep-alive policy applies only when there are more than corePoolSize threads. But method `allowCoreThreadTimeOut(boolean)` can be used to apply this time-out policy to core threads as well, so long as the keepAliveTime value is non-zero.**

**Queuing**
Any `BlockingQueue` may be used to transfer and hold submitted tasks. The use of this queue interacts with pool sizing:

* If fewer than **corePoolSize** threads are running, the `Executor` always prefers adding a new thread rather than queuing.
* If **corePoolSize** or more threads are running, the `Executor` always prefers queuing a request rather than adding a new thread.
* If a request cannot be queued, a new thread is created unless this would exceed **maximumPoolSize**, in which case`在这种情况下`, the task will be rejected.

There are three general strategies for queuing:

1. ***Direct handoffs传递***. A good default choice for a work queue is a `SynchronousQueue` that hands off tasks to threads without otherwise holding them. Here, an attempt to queue a task will fail if no threads are immediately available to run it, so a new thread will be constructed. This policy avoids lockups when handling sets of requests that might have internal dependencies. Direct handoffs generally require unbounded maximumPoolSizes to avoid rejection of new submitted tasks. This in turn admits the possibility of unbounded thread growth when commands continue to arrive on average faster than they can be processed.

2. ***Unbounded queues***. Using an unbounded queue (for example a `LinkedBlockingQueue` without a predefined capacity) will cause new tasks to wait in the queue when all corePoolSize threads are busy. Thus, **no more than corePoolSize threads will ever be created**. (And the value of the maximumPoolSize therefore doesn't have any effect.) This may be appropriate`合适` when each task is completely independent of others`完全独立于其他任务`, so tasks cannot affect each others execution`因此任务不能影响其他任务的执行`; for example, in a web page server. While this style of queuing can be useful in smoothing out transient bursts of requests`消除请求的短暂突发`, it admits the possibility of unbounded work queue growth when commands continue to arrive on average faster than they can be processed.`但它也承认当命令平均以高于处理速度的速度持续到达时，可能会出现无限制的工作队列增长`

3. ***Bounded queues***. A bounded queue (for example, an `ArrayBlockingQueue`) helps prevent resource exhaustion when used with finite maximumPoolSizes, but can be more difficult to tune`调整` and control. Queue sizes and maximum pool sizes may be traded off for each other: Using large queues and small pools minimizes CPU usage, OS resources, and context-switching overhead, but can lead to artificially low throughput. If tasks frequently block (for example if they are I/O bound), a system may be able to schedule time for more threads than you otherwise allow. Use of small queues generally requires larger pool sizes, which keeps CPUs busier but may encounter unacceptable scheduling overhead, which also decreases throughput.

   `队列大小和最大池大小可以相互权衡：使用大型队列和小型池可以最大限度地减少CPU使用、操作系统资源和上下文切换开销，但可能导致人为的低吞吐量。如果任务经常阻塞（例如，如果它们是I/O绑定的），系统可能能够为比您所允许的更多的线程安排时间。使用小队列通常需要更大的池大小，这样可以使CPU更忙，但可能会遇到不可接受的调度开销，这也会降低吞吐量。`

**Rejected tasks**
New tasks submitted in method `execute(Runnable)` will **be rejected when the Executor has been shut down, and also when the Executor uses finite bounds for both maximum threads and work queue capacity, and is saturated`饱和`**. In either case, the execute method invokes the `RejectedExecutionHandler.rejectedExecution(Runnable, ThreadPoolExecutor)` method of its `RejectedExecutionHandler`. Four predefined handler policies are provided:

1. In the default `ThreadPoolExecutor.AbortPolicy`, the handler throws a runtime `RejectedExecutionException` upon rejection.
2. In `ThreadPoolExecutor.CallerRunsPolicy`, the thread that **invokes execute itself runs the task**. This provides a simple feedback control mechanism that will **slow down the rate that new tasks are submitted**.
3. In `ThreadPoolExecutor.DiscardPolicy`, a task that cannot be executed is simply **dropped**.
4. In `ThreadPoolExecutor.DiscardOldestPolicy`, if the executor is not shut down, the task **at the head of the work queue is dropped**, and then execution is **retried** (which can fail again, causing this to be repeated.)


It is possible to define and use other kinds of `RejectedExecutionHandler` classes. Doing so requires some care especially when policies are designed to work only under particular capacity or queuing policies.`这样做需要一些注意，特别是当策略设计为仅在特定容量或排队策略下工作时`

**Hook methods**
This class provides protected overridable **beforeExecute(Thread, Runnable)** and **afterExecute(Runnable, Throwable)** methods that are called before and after execution of each task. These can be used to manipulate`操控` the execution environment; for example, reinitializing ThreadLocals, gathering statistics, or adding log entries. Additionally, method `terminated` can be overridden to perform any special processing that needs to be done once the `Executor` has fully terminated.
If hook or callback methods throw exceptions, internal worker threads may in turn fail and abruptly terminate.

**Queue maintenance`维修`**
**Method `getQueue()` allows access to the work queue for purposes of monitoring and debugging. Use of this method for any other purpose is strongly discouraged.** Two supplied methods, `remove(Runnable)` and `purge` are available to assist in storage reclamation when large numbers of queued tasks become cancelled.

**Finalization**
**A pool that is no longer referenced in a program "AND" <u>has no remaining threads</u> will be shutdown automatically**. If you would like to ensure that unreferenced pools are reclaimed even if users forget to call shutdown, then you **must arrange that unused threads eventually die, by setting appropriate keep-alive times, using a lower bound of zero core threads and/or setting allowCoreThreadTimeOut(boolean).**

Extension example. Most extensions of this class override one or more of the protected hook methods. For example, here is a subclass that adds a simple pause/resume feature:

```java
 class PausableThreadPoolExecutor extends ThreadPoolExecutor {
   private boolean isPaused;
   private ReentrantLock pauseLock = new ReentrantLock();
   private Condition unpaused = pauseLock.newCondition();

   public PausableThreadPoolExecutor(...) { super(...); }

   protected void beforeExecute(Thread t, Runnable r) {
     super.beforeExecute(t, r);
     pauseLock.lock();
     try {
       while (isPaused) unpaused.await();
     } catch (InterruptedException ie) {
       t.interrupt();
     } finally {
       pauseLock.unlock();
     }
   }

   public void pause() {
     pauseLock.lock();
     try {
       isPaused = true;
     } finally {
       pauseLock.unlock();
     }
   }

   public void resume() {
     pauseLock.lock();
     try {
       isPaused = false;
       unpaused.signalAll();
     } finally {
       pauseLock.unlock();
     }
   }
 }
```

