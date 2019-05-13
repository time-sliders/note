# Future

A `Future` represents the result of an asynchronous computation`计算`. Methods are provided to **check** if the computation is complete, to **wait** for its completion, and to **retrieve the result** of the computation. The result can only be retrieved using method `get` when the computation has completed, **blocking** if necessary until it is ready. **Cancellation** is performed by the `cancel` method. Additional methods are provided to determine if the task completed normally or was cancelled. Once a computation has completed, the computation cannot be cancelled. If you would like to use a `Future` for the sake of `为了`cancellability but not provide a usable result, you can declare types of the form Future<?> and return `null` as a result of the underlying`下层的` task.
**Sample Usage** (Note that the following classes are all made-up`编造的，不真实的`.)

```java
 interface ArchiveSearcher { String search(String target); }
 class App {
   ExecutorService executor = ...
   ArchiveSearcher searcher = ...
   void showSearch(final String target)
       throws InterruptedException {
     Future<String> future
       = executor.submit(new Callable<String>() {
         public String call() {
             return searcher.search(target);
         }});
     displayOtherThings(); // do other things while searching
     try {
       displayText(future.get()); // use future
     } catch (ExecutionException ex) { cleanup(); return; }
   }
 }
```


The `FutureTask` class is an implementation of `Future` that implements `Runnable`, and so may be executed by an `Executor`. For example, the above construction with submit could be replaced by:

```java
 FutureTask<String> future =
   new FutureTask<String>(new Callable<String>() {
     public String call() {
       return searcher.search(target);
   }});
 executor.execute(future);
```


Memory consistency effects: Actions taken by the asynchronous computation **happen-before** actions following the corresponding `Future.get()` in another thread.

#FutureTask

**A cancellable asynchronous computation.** This class provides a base implementation of `Future`, with methods to **start** and **cancel** a computation, **query** to see if the computation is complete, and **retrieve** the result of the computation. The result can only be retrieved when the computation has completed; **the get methods will block if the computation has not yet completed**. <u>Once the computation has completed, the computation cannot be restarted or cancelled</u> (unless the computation is invoked using `runAndReset`).
A `FutureTask` can be used to wrap a `Callable` or `Runnable` object. Because `FutureTask` implements `Runnable`, a `FutureTask` can be submitted to an `Executor` for execution.
In addition to serving as a standalone class, this class provides protected functionality that may be useful when creating customized task classes.

> Revision`修订` notes: This differs from previous versions of this class that relied on AbstractQueuedSynchronizer, mainly to avoid surprising users about retaining interrupt status during cancellation races. Sync control in the current design relies on a "state" field updated via CAS to track completion, along`以及` with a simple Treiber stack to hold waiting threads. Style note: As usual, we bypass overhead of using AtomicXFieldUpdaters and instead directly use Unsafe intrinsics.
>
> 修订说明：这与该类上一版本依赖AbstractQueuedSynchronizer不同，主要是为了避免用户在cancellation races期间对保留中断状态感到惊讶。当前设计中的同步控制依赖于通过CAS更新的 “state” 字段来跟踪完成情况，以及一个简单的 Treiber 栈来保存等待的线程。样式说明：和往常一样，我们绕过使用 AtomicXFieldUpdaters 的开销，而是直接使用 Unsafe 函数。

### Fields

```java
/**
 * The run state of this task, initially NEW.  The run state
 * transitions to a terminal state only in methods set,
 * setException, and cancel.  During completion, state may take on
 * transient values of COMPLETING (while outcome is being set) or
 * INTERRUPTING (only while interrupting the runner to satisfy a
 * cancel(true)). Transitions from these intermediate to final
 * states use cheaper ordered/lazy writes because values are unique
 * and cannot be further modified.
 *
 * Possible state transitions:
 * NEW -> COMPLETING -> NORMAL
 * NEW -> COMPLETING -> EXCEPTIONAL
 * NEW -> CANCELLED
 * NEW -> INTERRUPTING -> INTERRUPTED
 */
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;

/** The underlying callable; nulled out after running */
private Callable<V> callable;
/** The result to return or exception to throw from get() */
private Object outcome; // non-volatile, protected by state reads/writes
/** The thread running the callable; CASed during run() */
private volatile Thread runner;
/** Treiber stack of waiting threads */
private volatile WaitNode waiters;
```

其中 WaitNode 是一个简单的自定义单向链表，数据结构如下：

```java
/**
 * Simple linked list nodes to record waiting threads in a Treiber
 * stack.  See other classes such as Phaser and SynchronousQueue
 * for more detailed explanation.
 */
static final class WaitNode {
    volatile Thread thread;
    volatile WaitNode next;
    WaitNode() { thread = Thread.currentThread(); }
}
```

### 核心方法

####  get

```java
/**
 * @throws CancellationException {@inheritDoc}
 */
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);//等待异步计算结束
    return report(s);
}
```

```java
/**
 * Awaits completion or aborts on interrupt or timeout.
 *
 * @param timed true if use timed waits
 * @param nanos time to wait, if timed
 * @return state upon completion
 */
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
      	// 如果已经被外部中断，则直接结束等待
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }

        int s = state;
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        else if (q == null)
          	// 创建等待节点
            q = new WaitNode();
        else if (!queued)
          	// 入等待队列,这里是添加到队首(因为是单向链表，这样做更快)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        else if (timed) {
          	// 等待超时处理
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            LockSupport.parkNanos(this, nanos);
        }
        else
          	// 进入等待
            LockSupport.park(this);
    }
}
```

如果一个等待的线程被中断，则会从等待队列中移除，具体代码如下：

```java
/**
 * Tries to unlink a timed-out or interrupted wait node to avoid
 * accumulating garbage.  Internal nodes are simply unspliced
 * without CAS since it is harmless if they are traversed anyway
 * by releasers.  To avoid effects of unsplicing from already
 * removed nodes, the list is retraversed in case of an apparent
 * race.  This is slow when there are a lot of nodes, but we don't
 * expect lists to be long enough to outweigh higher-overhead
 * schemes.
 */
private void removeWaiter(WaitNode node) {
    if (node != null) {
	      // 设置成 null 之后，后面从链表中找到这个 thread 为 null 的 node 移除即可
        node.thread = null; 
        retry:
        for (;;) {          // restart on removeWaiter race
            for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                s = q.next; // 与上一行的 q = s 构成便利链表
                if (q.thread != null)
                    pred = q;
                else if (pred != null) {// 当前节点 thread 为 null，且上一个节点不为 null
                    pred.next = s;// 上一个节点的 next 重新指向当前节点的 next，即移除当前
                    if (pred.thread == null) // check for race
                        continue retry; // 如果上一个节点也被移除了，则重新处理
                }
              	// 如果是头节点被移除，则直接设置 waiters 为头节点的下一个节点即可
                else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                      q, s))
                    continue retry;
            }
          	// 当整个等待队列全部被重新处理了一遍，则接触循环
            break;
        }
    }
}
```

###  Run

```java
public void run() {
  	// 如果已经启动过了，则直接返回
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;// 是否是正常执行结束
            try {
                result = c.call();// 执行任务
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
              	// 执行结束，唤醒 get 等待线程
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

#### runAndReset

同 Run 类似，不过不会调用 set 方法唤醒等待线程，执行完毕之后，重置状态。

这个方法主要用在 不止一次使用的场景下

```java
/**
 * Executes the computation without setting its result, and then
 * resets this future to initial state, failing to do so if the
 * computation encounters an exception or is cancelled.  This is
 * designed for use with tasks that intrinsically本质上 execute more
 * than once.
 *
 * @return {@code true} if successfully run and reset
 */
protected boolean runAndReset() {
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return false;
    boolean ran = false;
    int s = state;
    try {
        Callable<V> c = callable;
        if (c != null && s == NEW) {
            try {
                c.call(); // don't set result
                ran = true;
            } catch (Throwable ex) {
                setException(ex);
            }
          	// 这里没有调用 set 唤醒 waitters
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
    return ran && s == NEW; // 正常执行完毕，则重置状态，以便下一次执行
}
```

#### set

set 会设置 Future 的状态，之后调用 finishCompletion 完成整个服务。

```java
/**
 * Sets the result of this future to the given value unless
 * this future has already been set or has been cancelled.
 *
 * <p>This method is invoked internally by the {@link #run} method
 * upon successful completion of the computation.
 *
 * @param v the value
 */
protected void set(V v) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state 终态
        finishCompletion();
    }
}
```

```java
/**
 * Removes and signals all waiting threads, invokes done(), and
 * nulls out callable.
 */
private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
      	// 移除等待队列
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                  	// 唤醒等待线程
                    LockSupport.unpark(t);
                }
              	// 继续处理后续节点
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }

  	/**
     * Protected method invoked when this task transitions to state
     * {@code isDone} (whether normally or via cancellation). The
     * default implementation does nothing.  Subclasses may override
     * this method to invoke completion callbacks or perform
     * bookkeeping记账. Note that you can query status inside the
     * implementation of this method to determine whether this task
     * has been cancelled.
     */
    done();

    callable = null;        // to reduce footprint
}
```

#### cancel

```java
public boolean cancel(boolean mayInterruptIfRunning) {
  	// 如果 state == NEW 则设置状态，并走后续逻辑，否则直接返回false ，不允许cancel
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
  	
    try {    // in case call to interrupt throws exception
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    t.interrupt();
            } finally { // final state
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        finishCompletion();
    }
    return true;
}
```