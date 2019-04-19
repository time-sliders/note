# Semaphore 信号量

## 介绍

Semaphore 是 Sync 一个实现，通过 state 字段存储 permit 许可证，由于 permit 可能是多个，所以存在多个线程同时获取到锁，任何一个 permit 的释放，都会 unpark 后续线程

## sourcecode

#### Sync 同步器

```java
/**
 * Synchronization implementation for semaphore.  Uses AQS state
 * to represent permits. Subclassed into fair and nonfair
 * versions.
 */
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 1192457210091910933L;

    Sync(int permits) {
        setState(permits); // 构造器初始化凭证总量
    }

    final int getPermits() {
        return getState();
    }

  	
    final int nonfairTryAcquireShared(int acquires) {
      	// for 循环不是自旋，只是在并发冲突之后的重试
        for (;;) {
            int available = getState();
            int remaining = available - acquires;
						// 如果remaining < 0 则表示没有资源，直接返回-1 表示失败
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }

    protected final boolean tryReleaseShared(int releases) {
        for (;;) {
            int current = getState();
            int next = current + releases;
            if (next < current) // overflow 不能释放负数
                throw new Error("Maximum permit count exceeded");
            if (compareAndSetState(current, next))
                return true;
        }
    }

  	// 自旋的方式减少 permit
    final void reducePermits(int reductions) {
        for (;;) {
            int current = getState();
            int next = current - reductions;
            if (next > current) // underflow
                throw new Error("Permit count underflow");
            if (compareAndSetState(current, next))
                return;
        }
    }

  	// 一次性拿完返回剩余数量
    final int drainPermits() {
        for (;;) {
            int current = getState();
            if (current == 0 || compareAndSetState(current, 0))
                return current;
        }
    }
}
```

#### NonfairSync

```java
/**
 * NonFair version
 */
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -2694183684443567898L;

    NonfairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        return nonfairTryAcquireShared(acquires);
    }
}
```

不公平锁直接调用 Sync 不公平申请资源即可

#### FairSync

```java
/**
 * Fair version
 */
static final class FairSync extends Sync {
    private static final long serialVersionUID = 2014338818796000944L;

    FairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        for (;;) {
            if (hasQueuedPredecessors()) // 如果有等待队列，则入队
                return -1;
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
}
```

公平锁在尝试之前，先判断有无等待，有的话，直接入队

#### 核心对外方法

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
public boolean tryAcquire() {
    return sync.nonfairTryAcquireShared(1) >= 0;
}
public void release() {
    sync.releaseShared(1);
}
```

## javadoc

A counting semaphore`用来计数的`. Conceptually`概念地`, a `semaphore` maintains a set of **permits**. Each `acquire` blocks if necessary until a **permit** is available, and then takes it. Each `release` adds a **permit**, potentially`潜在地` releasing a blocking acquirer. However, no actual permit objects are used`然而，并没有使用真实的permit对象`; the `Semaphore` just **keeps a count of the number available and acts accordingly**`依据`.
`Semaphores` are often used to restrict`约束` the number of threads than can access some (physical or logical) resource. For example, here is a class that uses a semaphore to control access to a pool of items:

```java
 class Pool {
   private static final int MAX_AVAILABLE = 100;
   private final Semaphore available = new Semaphore(MAX_AVAILABLE, true);

   public Object getItem() throws InterruptedException {
     available.acquire();
     return getNextAvailableItem();
   }

   public void putItem(Object x) {
     if (markAsUnused(x))
       available.release();
   }

   // Not a particularly efficient data structure; just for demo

   protected Object[] items = ... whatever kinds of items being managed
   protected boolean[] used = new boolean[MAX_AVAILABLE];

   protected synchronized Object getNextAvailableItem() {
     for (int i = 0; i < MAX_AVAILABLE; ++i) {
       if (!used[i]) {
         used[i] = true;
         return items[i];
       }
     }
     return null; // not reached
   }

   protected synchronized boolean markAsUnused(Object item) {
     for (int i = 0; i < MAX_AVAILABLE; ++i) {
       if (item == items[i]) {
         if (used[i]) {
           used[i] = false;
           return true;
         } else
           return false;
       }
     }
     return false;
   }
 }
```

Before obtaining an item each thread must acquire a **permit** from the semaphore, guaranteeing that an item is available for use. When the thread has finished with the item it is returned back to the pool and a permit is returned to the semaphore, allowing another thread to acquire that item. Note that`注意` no synchronization lock is held when `acquire` is called as that`那样` would prevent an item from being returned to the pool. The semaphore encapsulates`封装` the synchronization needed`需求` to restrict access to the pool, separately from any synchronization needed to maintain the consistency`一致性` of the pool itself.
A `semaphore` initialized to **one**, and which is used such that it only has at most one permit available, **can serve as a mutual exclusion lock**. This is more commonly known as a **binary semaphore**, because it only has two states: one permit available, or zero permits available. When used in this way, the binary semaphore has the property (unlike many `java.util.concurrent.locks.Lock` implementations), **that the "lock" can be released by a thread other than the owner** (as semaphores have no notion of ownership`没有所有权的概念`). This can be useful in some specialized contexts, such as **deadlock recovery**.
The constructor for this class optionally`可选择的` accepts a **fairness** parameter. When set false, this class makes no guarantees about the order in which threads acquire permits. In particular`特别的`, barging is permitted, that is, a thread invoking acquire can be allocated a permit ahead of a thread that has been waiting - logically the new thread places itself at the head of the queue of waiting threads. When **fairness** is set true, the semaphore guarantees that threads invoking **any** of the acquire methods are selected to obtain permits in the order in which their invocation of those methods was processed (first-in-first-out; FIFO). Note that FIFO ordering necessarily applies to specific internal points of execution within these methods. So, it is possible for one thread to invoke `acquire` before another, but reach the ordering point after the other, and similarly upon `return` from the method. Also note that the untimed`无超时的` `tryAcquire` methods do not honor`遵守` the fairness setting, but will take any permits that are available.
Generally, semaphores used to **control resource access should be initialized as fair**, to ensure that no thread is starved out`饿死` from accessing a resource. When using semaphores for other kinds of synchronization control, the throughput advantages`益处` of non-fair ordering often outweigh`比...有价值` fairness considerations.
This class also provides convenience methods to `acquire` and `release` multiple permits at a time. These methods are generally more efficient and effective than loops. However, they do not establish any preference order. For example, if thread A invokes s.acquire(3) and thread B invokes s.acquire(2), and two permits become available, then there is no guarantee that thread B will obtain them unless its acquire came first and `Semaphore` s is in fair mode.
Memory consistency effects: Actions in a thread prior to calling a "`release`" method such as `release()` *happen-before* actions following a successful "acquire" method such as acquire() in another thread.