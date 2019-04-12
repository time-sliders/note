# AbstractQueuedSynchronizer (AQS锁)

## 介绍

一个在 LockSupport 的基础上实现的队列同步器，内部维护了维护了一个volatile int state（代表共享资源）和一个FIFO的等待队列，支持独占/共享两种模式。由于 tryAcquire 方法是先检查，再入队。所以在极端场景下，存在一个会导致不公平的现象，比如 A 线程持有锁，B 线程申请时，发现已经被 A 独占，所以 B 执行入队，再 B 还未完成入队的情况下，C 线程正好申请锁，而且恰好此时 A 线程释放了这把锁，那么这个时候，C线程有可能比 B 线程 先拿到锁，从而导致不公平的现象，这个不公平的机制可以提高锁的吞吐量和可伸缩性，性能更好，所以是默认的策略。

AQS 队列同步器只实现锁队列&中断唤醒等方法，对于共享资源的管理，需要子类实现。

## 源码解析

### acquire

```java
/**
 * Acquires in exclusive mode, ignoring interrupts.  Implemented
 * by invoking at least once {@link #tryAcquire},
 * returning on success.  Otherwise the thread is queued, possibly
 * repeatedly blocking and unblocking, invoking {@link
 * #tryAcquire} until success.  This method can be used
 * to implement method {@link Lock#lock}.
 *
 * @param arg the acquire argument.  This value is conveyed传递 to
 *        {@link #tryAcquire} but is otherwise uninterpreted and
 *        can represent anything you like.
 */
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

该方法定义了独占模式下，获取共享资源的方式。如果获取到资源，就会直接返回，否组会进入等待队列，直到获取到资源为止。整个过程忽略中断的影响，这个方法可以用来实现 lock 语义。

函数流程如下：

1. `tryAcquire()` 尝试直接去获取资源，如果成功则直接返回；
2. `addWaiter()` 将该线程加入等待队列的尾部，并标记为独占模式；
3. `acquireQueued()` **使线程在等待队列中休息，有机会时（轮到自己，会被 `unpark()`）会去尝试获取资源。获取到资源后才返回。**如果在整个等待过程中被中断过，则返回true，否则返回false。
4. 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断`selfInterrupt()`，将中断补上。

#### acquireQueued

该方法是申请锁时，进入等待的核心逻辑。

```java
/**
 * Acquires in exclusive uninterruptible mode for thread already in
 * queue. Used by condition wait methods as well as acquire.
 *
 * @param node the node
 * @param arg the acquire argument
 * @return {@code true} if interrupted while waiting
 */
final boolean acquireQueued(final Node node, int arg) {
    try {
        boolean interrupted = false;
        for (;;) {
          	// 拿到前一个节点 p
            final Node p = node.predecessor();
          	/**
          	 * 只有当前一个节点是 head 的时候才尝试去获取锁，
          	 * 因为此时，前一个节点，随时可能会释放锁
          	 * 另外也可以看出来处于队列中 Waiter 拿到锁的顺序是公平的
          	 */
            if (p == head && tryAcquire(arg)) {
              	// 拿到锁之后，设置节点为 head，所以 head 也表示获取到锁的节点
                setHead(node);
                p.next = null; // help GC 前一个节点出队，可以被 GC 回收了
                return interrupted;
            }
            // 判断是否需要 park 
            if (shouldParkAfterFailedAcquire(p, node) &&
                // 调用 LockSupport->Unsafe#park 线程中断
                parkAndCheckInterrupt()) 
                interrupted = true;
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}
```

#### shouldParkAfterFailedAcquire

该方法判断前一个节点是否会在处理完之后通知自己，如果会，则进入 park 等待，否则告诉前一个节点处理完要通知自己，然后再进入 park。

```java
/**
 * Checks and updates status for a node that failed to acquire.
 * Returns true if thread should block. This is the main signal
 * control in all acquire loops.  Requires that pred == node.prev.
 *
 * @param pred node's predecessor holding status
 * @param node the node
 * @return {@code true} if thread should block
 */
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         * 上一个节点已经设置状态为：在释放锁之后，通知当前节点。所以现在可以安全的park
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over略过 predecessors and
         * indicate retry.
         * 如果上一个节点放弃了，那就一直往前找，直到找到最近一个正常等待的状态，并排在它的后边
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         * 如果前一个节点正常，那就把它的状态设置成SIGNAL，告诉它拿完号后通知自己一下
         * 这里有可能是失败，因为前一个节点有可能刚刚改过状态
         */
        pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
    }
    return false;
}
```

#### tryAcquire

尝试获取独占锁，如果获取成功，返回 true，否则返回 false。**由于 AQS 只是一个框架，所以这里获取资源的具体的方式，需要由子类自己去实现**。

```java
/**
 * Attempts to acquire in exclusive mode. This method should query
 * if the state of the object permits it to be acquired in the
 * exclusive mode, and if so to acquire it.
 *
 * <p>This method is always invoked by the thread performing
 * acquire.  If this method reports failure, the acquire method
 * may queue the thread, if it is not already queued, until it is
 * signalled by a release from some other thread. This can be used
 * to implement method {@link Lock#tryLock()}.
 *
 * <p>The default
 * implementation throws {@link UnsupportedOperationException}.
 *
 * @param arg the acquire argument. This value is always the one
 *        passed to an acquire method, or is the value saved on entry
 *        to a condition wait.  The value is otherwise uninterpreted
 *        and can represent anything you like.
 * @return {@code true} if successful. Upon success, this object has
 *         been acquired.
 * @throws IllegalMonitorStateException if acquiring would place this
 *         synchronizer in an illegal state. This exception must be
 *         thrown in a consistent fashion for synchronization to work
 *         correctly.
 * @throws UnsupportedOperationException if exclusive mode is not supported
 */
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

**综上** 整个申请锁的流程为

![AQS-Acquire](/Users/zhangwei/github/note/java/lock/ref/juc-AQS-Acquire.png)



### release

在独占模式下，释放所占用的资源。根据实现来释放一个或多个资源。该方法一般用来实现 unlock 语义。

```java
/**
 * Releases in exclusive mode.  Implemented by unblocking one or
 * more threads if {@link #tryRelease} returns true.
 * This method can be used to implement method {@link Lock#unlock}.
 *
 * @param arg the release argument.  This value is conveyed to
 *        {@link #tryRelease} but is otherwise uninterpreted and
 *        can represent anything you like.
 * @return the value returned from {@link #tryRelease}
 */
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head; // 找到头节点
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h); // 唤醒下一个线程
        return true;
    }
    return false;
}
```

注意这里是根据子类定义的 `tryRelease` 具体实现，来释放资源。由于是独占模式，释放的线程已经成为了head 拿到了资源，所以释放一般会成功。这里根据  `tryRelease` 的返回值来判断是否释放成功，所以子类需要根据实际释放结果返回正确的值。

#### unparkSuccessor

该方法用户唤醒等待队列中的下一个线程

```java
/**
 * Wakes up node's successor继承人, if one exists.
 *
 * @param node the node
 */
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        node.compareAndSetWaitStatus(ws, 0);// 置0当前节点的状态

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail(从尾部向后移动) to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;// 下一个需要唤醒的节点
    if (s == null || s.waitStatus > 0) {// 为空或者已经取消
        s = null;
        // 如果下一个节点是无效的，那么重新从 tail 往前查找，直到找到有效节点
        for (Node p = tail; p != node && p != null; p = p.prev)
            if (p.waitStatus <= 0)// < 0 的节点都是有效的节点
                s = p; // 这里没有 break 所以会一直往前，直到找到最前面的那个有效节点
    }
    if (s != null)
        LockSupport.unpark(s.thread);//唤醒
}
```

这个函数并不复杂。一句话概括：**用`unpark()`唤醒等待队列中最前边的那个未放弃线程**，即 s。此时，再和`acquireQueued()`联系起来，s 被唤醒后，进入 `if (p == head && tryAcquire(arg))` 的判断（即使`p!=head`也没关系，它会再进入 `shouldParkAfterFailedAcquire()` 寻找一个安全点。这里既然 s 已经是等待队列中最前边的那个未放弃线程了，那么通过`shouldParkAfterFailedAcquire()`的调整，s 也必然会跑到`head`的`next`结点，下一次自旋`p==head`就成立啦），然后 s 把自己设置成`head`结点，表示自己已经获取到资源了，`acquire()`也返回了

### acquireShared

此方法是共享模式下线程获取共享资源的顶层入口。它会获取**指定量**的资源，获取成功则直接返回，获取失败则进入等待队列，直到获取到资源为止，整个过程忽略中断

```java
/**
 * Acquires in shared mode, ignoring interrupts.  Implemented by
 * first invoking at least once {@link #tryAcquireShared},
 * returning on success.  Otherwise the thread is queued, possibly
 * repeatedly blocking and unblocking, invoking {@link
 * #tryAcquireShared} until success.
 *
 * @param arg the acquire argument.  This value is conveyed to
 *        {@link #tryAcquireShared} but is otherwise uninterpreted
 *        and can represent anything you like.
 */
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

这里 `tryAcquireShared()` 依然需要自定义同步器去实现。但是 AQS 已经把其返回值的语义定义好了：**负值代表获取失败；0代表获取成功，但没有剩余资源；正数表示获取成功，还有剩余资源，其他线程还可以去获取**。所以这里 `acquireShared()` 的流程就是：

1. `tryAcquireShared()` 尝试获取资源，成功则直接返回。
2. 失败则通过 `doAcquireShared()` 进入等待队列，直到获取到资源为止才返回。

#### doAcquireShared

```java
/**
 * Acquires in shared uninterruptible mode.
 * @param arg the acquire argument
 */
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED); // 入队
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) { // head是拿到资源的线程，head用完资源来唤醒自己
                int r = tryAcquireShared(arg);
                if (r >= 0) { // >=0 成功，正数表示还有剩余资源
                    setHeadAndPropagate(node, r);// 升为head，如果有多余的资源，传播下去
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    return;
                }
            }
          
          	//判断状态，寻找安全点，进入 park() 状态，等着被 unpark() 或 interrupt()
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}
```

同 `acquireQueued()` 很相似，其实流程并没有太大区别。区别是，如果获取到资源之后，还有剩余，那么会继续传播给后续线程，从而实现共享。

#### setHeadAndPropagate

```java
/**
 * Sets head of queue, and checks if successor may be waiting
 * in shared mode, if so propagating if either propagate > 0 or
 * PROPAGATE status was set.
 *
 * @param node the node
 * @param propagate the return value from a tryAcquireShared
 */
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node); // 将 head 指向自己
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus either before
     *     or after setHead) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition转换/过渡 to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     * 
     * 如果还有剩余，则唤醒后续线程 （propagate > 0 表示还有剩余）
     */
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

## javadoc 翻译

Provides a framework for implementing blocking locks and related synchronizers (semaphores, events, etc) that rely on first-in-first-out (FIFO) wait queues`提供了一种实现阻塞锁和一系列依赖FIFO等待队列的同步器的框架`. This class is designed to be a useful basis for most kinds of synchronizers that **rely on a single atomic int value to represent state**. Subclasses must define the `protected` methods that change this **state**, and which define what that **state** means in terms of`依据` this object being *acquired* or *released*. Given these, the other methods in this class carry out`执行` all queuing and blocking mechanics`机制`. Subclasses can maintain other state fields, but only the atomically updated int value manipulated using methods `getState`, `setState` and `compareAndSetState` is tracked with respect to synchronization.

**Subclasses should be defined as non-public internal helper classes that are used to implement the synchronization properties of their enclosing`内部` class**. Class `AbstractQueuedSynchronizer` does not implement any synchronization interface. Instead it defines methods such as `acquireInterruptibly` that can be invoked as appropriate`适当的` by concrete`具体的` locks and related synchronizers to implement their public methods`相反，它定义了AcquireInterruptly等方法，这些方法可以由具体的锁和相关的同步器根据需要调用，以实现它们的公共方法`。.

This class supports either or both a default exclusive`独占` mode and a shared`共享` mode`此类支持默认独占模式和共享模式中的一种或两种`. When acquired in exclusive mode, attempted acquires by other threads cannot succeed. Shared mode acquires by multiple threads may (but need not`但不需要`) succeed. This class does not "understand" these differences except in the mechanical sense that when a shared mode acquire succeeds, the next waiting thread (if one exists) must also determine whether it can acquire as well. **Threads waiting in the different modes share the same FIFO queue**. Usually, implementation subclasses support only one of these modes, but both can come into play for example in a `ReadWriteLock`. Subclasses that support only exclusive or only shared modes need not define the methods supporting the unused mode.

This class defines a nested `ConditionObject` class that can be used as a `Condition` implementation by subclasses supporting exclusive mode for which method `isHeldExclusively` reports whether synchronization is exclusively held with respect to the current thread, method `release` invoked with the current `getState` value fully releases this object, and `acquire`, given this saved state value, eventually restores this object to its previous acquired state. No `AbstractQueuedSynchronizer` method otherwise creates such a condition, so if this constraint cannot be met, do not use it. The behavior of `ConditionObject` depends of course on the semantics`语义` of its synchronizer implementation.

This class provides inspection`检查`, instrumentation`仪表`, and monitoring methods for the internal queue, as well as similar methods for condition objects. These can be exported as desired into classes using an `AbstractQueuedSynchronizer` for their synchronization mechanics.`这个可以被期望用在实现子类基于 AQS 的同步机制`

Serialization of this class stores only the underlying atomic integer maintaining state, so deserialized objects have empty thread queues. Typical subclasses requiring serializability will define a readObject method that restores this to a known initial state upon deserialization.

**Usage**
To use this class as the basis of a synchronizer, redefine the following methods, as applicable`可应用的`, by inspecting and/or modifying the synchronization state using `getState`, `setState` and/or `compareAndSetState`:

* `tryAcquire`
* `tryRelease`
* `tryAcquireShared`
* `tryReleaseShared`
* `isHeldExclusively`

Each of these methods by default throws *UnsupportedOperationException*. Implementations of these methods must be internally thread-safe, and should in general be short and not block. Defining these methods is the **only** supported means of using this class. All other methods are declared `final` because they cannot be independently varied`多变的`.

You may also find the inherited methods from `AbstractOwnableSynchronizer` useful to keep track of the thread owning an exclusive synchronizer. You are encouraged to use them -- this enables monitoring and diagnostic tools to assist users in determining which threads hold locks.

Even though this class is based on an internal FIFO queue, it **does not automatically enforce`强制执行` FIFO acquisition policies**. The core of exclusive synchronization takes the form:
   Acquire:

```java
while (!tryAcquire(arg)) {
  enqueue thread if it is not already queued;
  possibly block current thread;
}
```

   Release:

```java
 if (tryRelease(arg))
          unblock the first queued thread;
```

(Shared mode is similar but may involve cascading signals`涉及级联信号`.)
**Because checks in acquire are invoked before enqueuing`入队`, a newly acquiring thread may barge`闯入` ahead of others that are blocked and queued.** However, you can, if desired, define `tryAcquire` and/or `tryAcquireShared` to disable barging by internally invoking one or more of the inspection`检查` methods, thereby`从而` providing a **fair** FIFO acquisition order. In particular, most fair synchronizers can define `tryAcquire` to return false if `hasQueuedPredecessors` (a method specifically designed to be used by fair synchronizers) returns true. Other variations`变化` are possible.

Throughput`吞吐量` and scalability`可扩展性` are generally highest for the default barging (also known as *greedy*`贪心`, *renouncement*`拒绝`, and *convoy-avoidance*`避孕`) strategy. While`虽然` this is not guaranteed to be fair or starvation-free`无饥饿`, earlier queued threads are allowed to recontend`再竞争` before later queued threads, and each recontention has an unbiased`无偏见的` chance to succeed against`在…之前` incoming threads. Also`此外`, while`虽然` acquires do not "**spin**" in the usual sense, they may perform multiple invocations of `tryAcquire` interspersed`多次` with other computations`计算/估计` before blocking. This gives most of the benefits of spins when exclusive synchronization is only briefly held, without most of the liabilities when it isn't. If so desired, you can augment this by preceding calls to `acquire` methods with "fast-path" checks, possibly prechecking `hasContended` and/or `hasQueuedThreads` to only do so if the synchronizer is likely not to be contended.

**This class provides an efficient and scalable basis for synchronization in part by specializing its range of use to synchronizers that can rely on int state, acquire, and release parameters, and an internal FIFO wait queue**. When this does not suffice`满足要求`, you can build synchronizers from a lower level using atomic classes, your own custom *java.util.Queue* classes, and *LockSupport* blocking support.
**Usage Examples**
Here is a non-reentrant mutual`相互` exclusion lock class that uses the value *zero* to represent the unlocked state, and *one* to represent the locked state. While a non-reentrant lock does not strictly require`严格要求` recording of the current owner thread, this class does so anyway`依然这样做了` to make usage easier to monitor. It also supports `conditions` and exposes one of the instrumentation methods:

 `Mutex => mutual exclusion 互斥`

```java
class Mutex implements Lock, java.io.Serializable {

   // Our internal helper class
   private static class Sync extends AbstractQueuedSynchronizer {
     // Reports whether in locked state
     protected boolean isHeldExclusively() {
       return getState() == 1;
     }

     // Acquires the lock if state is zero
     public boolean tryAcquire(int acquires) {
       assert acquires == 1; // Otherwise unused
       if (compareAndSetState(0, 1)) {
         setExclusiveOwnerThread(Thread.currentThread());
         return true;
       }
       return false;
     }

     // Releases the lock by setting state to zero
     protected boolean tryRelease(int releases) {
       assert releases == 1; // Otherwise unused
       if (getState() == 0) throw new IllegalMonitorStateException();
       setExclusiveOwnerThread(null);
       setState(0);
       return true;
     }

     // Provides a Condition
     Condition newCondition() { return new ConditionObject(); }

     // Deserializes properly
     private void readObject(ObjectInputStream s)
         throws IOException, ClassNotFoundException {
       s.defaultReadObject();
       setState(0); // reset to unlocked state
     }

   }
  
 	// The sync object does all the hard work. We just forward to it.
   private final Sync sync = new Sync();

   public void lock()                { sync.acquire(1); }
   public boolean tryLock()          { return sync.tryAcquire(1); }
   public void unlock()              { sync.release(1); }
   public Condition newCondition()   { return sync.newCondition(); }
   public boolean isLocked()         { return sync.isHeldExclusively(); }
   public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
   public void lockInterruptibly() throws InterruptedException {
     sync.acquireInterruptibly(1);
   }
   public boolean tryLock(long timeout, TimeUnit unit)
       throws InterruptedException {
     return sync.tryAcquireNanos(1, unit.toNanos(timeout));
   }
 }
```


Here is a latch`闩` class that is like a *CountDownLatch* except that it only requires a single signal to fire. Because a latch is non-exclusive, it uses the shared acquire and release methods.

``` java
class BooleanLatch {

   private static class Sync extends AbstractQueuedSynchronizer {
     boolean isSignalled() { return getState() != 0; }

     protected int tryAcquireShared(int ignore) {
       return isSignalled() ? 1 : -1;
     }

     protected boolean tryReleaseShared(int ignore) {
       setState(1);
       return true;
     }
   }

   private final Sync sync = new Sync();
   public boolean isSignalled() { return sync.isSignalled(); }
   public void signal()         { sync.releaseShared(1); }
   public void await() throws InterruptedException {
     sync.acquireSharedInterruptibly(1);
   }
 }
```



 

