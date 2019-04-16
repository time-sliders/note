# ReentrantLock 重入锁

## 介绍

在 AQS 锁的基础上，实现了可重入的实现（如果一个线程已经获取到了重入锁，那么这个线程可以多次进入这个锁），另外，在 AQS 默认不公平锁的基础之上，实现了公平锁的策略，但是不建议使用，吞吐量会低一点。

#### 优点 & 使用场景

`ReentrantLock` 可以等同于 `synchronized` 使用，但是比 `synchronized` 有更强的功能、可以提供更灵活的锁机制（比如**时间锁等候**、**可中断锁等候**、**无块结构锁**、**多个条件变量**或者**轮询锁**），**减少死锁的发生概率**。

在并发量较小的多线程应用程序中，`ReentrantLock` 与 `synchronized` 性能相差无几，但在高并发量的条件下，`synchronized` 性能会迅速下降几十倍，而 `ReentrantLock` 的性能却能依然维持一个水准，**应当在高度争用的情况下使用它**。

`ReentrantLock`通过方法 `lock()` 与 `unlock()` 来进行加锁与解锁操作，与 **`synchronized` 会被JVM自动解锁**机制不同，**`ReentrantLock` 加锁后需要手动进行解锁**。为了避免程序出现异常而无法正常解锁的情况，使用**`ReentrantLock` 必须在 `finally` 控制块中进行解锁操作**。

## 源码分析

#### Sync

```java
/** Synchronizer providing all implementation mechanics */
private final Sync sync;

/**
 * Base of synchronization control for this lock. Subclassed
 * into fair and nonfair versions below. Uses AQS state to
 * represent the number of holds on the lock.
 */
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;

    /**
     * Performs non-fair tryLock.  tryAcquire is implemented in
     * subclasses, but both need nonfair try for trylock method.
     */
    @ReservedStackAccess
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

    @ReservedStackAccess
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }

    protected final boolean isHeldExclusively() {
        // While we must in general read state before owner,
        // we don't need to do so to check if current thread is owner
        return getExclusiveOwnerThread() == Thread.currentThread();
    }

    final ConditionObject newCondition() {
        return new ConditionObject();
    }

    // Methods relayed from outer class

    final Thread getOwner() {
        return getState() == 0 ? null : getExclusiveOwnerThread();
    }

    final int getHoldCount() {
        return isHeldExclusively() ? getState() : 0;
    }

    final boolean isLocked() {
        return getState() != 0;
    }

    /**
     * Reconstitutes the instance from a stream (that is, deserializes it).
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        setState(0); // reset to unlocked state
    }
}
```

`ReentrantLock` 内部是引用 AQS 锁机制，state 表示的是当前 lock 被单个线程调用的次数，也就是重入次数，不能超过最大值。提供了一个 `nonfairTryAcquire` 的方法，用以提供 不公平的锁。该方法内部，在没有被其他线程锁的情况下，所有新加入的线程都是先通过 CAS 操作获取锁，忽略 AQS 的 WaiterQueue，所以不公平。



#### NonfairSync

```java
/**
 * Sync object for non-fair locks
 */
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

不公平锁的 `tryAcquire` 方法，是直接调用 Sync 的 `nonfairTryAcquire` 实现的不公平方式

####FairSync

```java
/**
 * Sync object for fair locks
 */
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;
    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    @ReservedStackAccess
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

公平锁在尝试获取锁之前，先判断是否有等待队列 `hasQueuedPredecessors` 如果有的话，就加锁失败，后续会在 AQS 的 `acquire` 方法里面，会进行 `enqueue` 排队处理。

#### constructor

```java
/**
 * Creates an instance of {@code ReentrantLock}.
 * This is equivalent to using {@code ReentrantLock(false)}.
 */
public ReentrantLock() {
    sync = new NonfairSync();// 默认是非公平锁
}

/**
 * Creates an instance of {@code ReentrantLock} with the
 * given fairness policy.
 *
 * @param fair {@code true} if this lock should use a fair ordering policy
 */
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

## javadoc

A reentrant`可重入的` mutual exclusion`互斥` ***Lock*** with the same basic behavior and semantics`语义` as the implicit`内置的` monitor lock accessed using `synchronized` methods and statements, but with extended capabilities.

A `ReentrantLock` is **owned** by the thread last successfully locking, but not yet unlocking it. A thread invoking `lock` will return, successfully acquiring the lock, when the lock is not owned by another thread. <u>The method will return immediately if the current thread already owns the lock</u>. This can be checked using methods `isHeldByCurrentThread`, and `getHoldCount`.

The constructor for this class accepts an optional **fairness** parameter. When set **true**, under contention`在争抢的场景下`, locks favor`支持` granting`准许` access to the longest-waiting thread. Otherwise this lock does not guarantee`保证` any particular`特定` access order. Programs using fair locks accessed by many threads may display lower overall throughput`表现出低一点的整体吞吐量` (i.e., are slower; often much slower`比较慢，通常要慢得多`) than those using the default setting, but have smaller variances`差异` in times to obtain locks and guarantee lack of starvation`无饥饿性`. Note however`但是请注意`, that **fairness of locks does not guarantee fairness of thread scheduling**. Thus, one of many threads using a fair lock may obtain it multiple times in succession while other active threads are not progressing and not currently holding the lock. Also note that the **untimed `tryLock()` method does not honor`支持` the fairness setting**. It will succeed if the lock is available even if other threads are waiting.
It is recommended practice to always immediately follow a call to lock with a try block, most typically in a before/after construction such as:

```java
 class X {
   private final ReentrantLock lock = new ReentrantLock();
   // ...

   public void m() {
     lock.lock();  // block until condition holds
     try {
       // ... method body
     } finally {
       lock.unlock()
     }
   }
 }
```

In addition to implementing the `Lock` interface, this class defines a number of public and protected methods for inspecting the state of the lock. Some of these methods are only useful for *instrumentation* and *monitoring*.
Serialization of this class behaves in the same way as built-in locks: a deserialized lock is in the unlocked state, regardless of its state when serialized.
This lock supports a maximum of 2147483647 recursive`递归` locks by the same thread. Attempts to exceed`超出` this limit result in Error throws from *locking* methods.