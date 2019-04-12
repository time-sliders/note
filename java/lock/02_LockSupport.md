# LockSupport

#### 介绍

通过 Unsafe 类的底层方法，实现了线程 park 和 unpark 的逻辑。这个类主要用于给其他高级的锁，提供锁实现的一些基础方法。
## javadoc 翻译

Basic thread blocking primitives for creating locks and other synchronization classes. 

**LockSupport 是用来创建锁和其他同步类的基本线程阻塞原语。**

This class associates`与……发生联系`, with each thread that uses it, a **permit** (in the sense`感觉` of the *Semaphore* class). A call to `park` will return immediately if the **permit** is available, consuming it in the process; otherwise it *may* block. A call to `unpark` makes the **permit** available, if it was not already available. (Unlike with Semaphores though`虽然`, **permits** do not accumulate. <u>There is at most one</u>.) 

Reliable usage requires the use of `volatile` (or `atomic`) variables to control when to `park` or `unpark`. Orderings of calls`调用顺序` to these methods are maintained`维护` with respect to`关于` volatile variable accesses, but not necessarily `non-volatile` variable accesses.

Methods `park` and `unpark` provide efficient means of`有效的方法` blocking and unblocking threads that do not encounter`遭遇` the problems that cause the deprecated methods `Thread.suspend` and `Thread.resume` to be unusable for such purposes: Races between one thread invoking `park` and another thread trying to `unpark` it will preserve liveness`保持活力`, due to the **permit**. Additionally`另外`, `park` will return if the caller's thread was interrupted, and timeout versions are supported. The `park` method may also return at any other time, for "no reason", <u>so in general must be invoked within a loop that rechecks conditions upon return</u>. In this sense `park` serves as`充当` an optimization`优化` of a "busy wait"`忙碌等待` that does not waste`浪费` as much time spinning`自旋`, but must be paired with an `unpark` to be effective.

The three forms`形式` of `park` each also support a `blocker` object parameter. This object is recorded while the thread is blocked to permit monitoring and diagnostic tools to identify the reasons that threads are blocked`当线程被阻塞时会记录此对象，以允许监视和诊断工具识别线程被阻塞的原因`. (Such tools may access blockers using method `getBlocker(Thread)`.) <u>The use of these forms rather than the original forms without this parameter is strongly encouraged</u>. <u>The normal argument to supply as a blocker within a lock implementation is **this**.</u>
These methods are designed to be used as tools for creating higher-level synchronization utilities, and are not in themselves useful for most concurrency control applications`这些方法被设计成用于创建更高级别的同步实用程序的工具，它们本身对于大多数并发控制应用程序都不有用`. The `park` method is designed for use only in constructions of the form:

```java
 while (!canProceed()) {
   // ensure request to unpark is visible to other threads
   ...
   LockSupport.park(this);
 }
```

where no actions by the thread publishing a request to `unpark`, prior to the call to park`如果线程在调用park之前没有发布unpark请求的操作`, entail`使产生` locking or blocking. Because only one **permit** is associated with each thread, any intermediary`中间阶段` uses of park, including implicitly`隐式的` via class loading, could lead to an unresponsive thread (a "lost unpark").
Sample Usage. Here is a sketch of a first-in-first-out non-reentrant lock class:

```java
 class FIFOMutex {
   private final AtomicBoolean locked = new AtomicBoolean(false);
   private final Queue<Thread> waiters
     = new ConcurrentLinkedQueue<>();

   public void lock() {
     boolean wasInterrupted = false;
     // publish current thread for unparkers
     waiters.add(Thread.currentThread());

     // Block while not first in queue or cannot acquire lock
     while (waiters.peek() != Thread.currentThread() ||
            !locked.compareAndSet(false, true)) {
       LockSupport.park(this);
       // ignore interrupts while waiting
       if (Thread.interrupted())
         wasInterrupted = true;
     }

     waiters.remove();
     // ensure correct interrupt status on return
     if (wasInterrupted)
       Thread.currentThread().interrupt();
   }

   public void unlock() {
     locked.set(false);
     LockSupport.unpark(waiters.peek());
   }

   static {
     // Reduce the risk of "lost unpark" due to classloading
     Class<?> ensureLoaded = LockSupport.class;
   }
 }
```
## 许可
*LockSupport* 通过许可 **permit** 实现挂起线程、唤醒挂起线程功能。可以按照以下逻辑理解：

- `pack` 时
  如果线程的 **permit** 存在，那么线程不会被挂起，立即返回；如果线程的 **permit** 不存在，认为线程缺少 **permit**，所以需要挂起等待 **permit**。

- `unpack` 时
  如果线程的 **permit** 不存在，那么释放一个 **permit**。因为有 **permit** 了，所以如果线程处于挂起状态，那么此线程会被线程调度器唤醒。如果线程的 **permit** 存在，**permit** 也不会累加 (最多只允许一个 **permit** 存在)，看起来想什么事都没做一样。注意这一点和 *Semaphore* 是不同的。

## 源码分析

#### park 主要功能

如果许可存在，那么将这个许可使用掉，并且立即返回。如果许可不存在，那么挂起当前线程，直到以下任意一件事情发生：

- 其他线程以当前线程为参数，调用 `unpark(thread)` 方法
- 其他线程通过 `Thread#interrupt` 中断当前线程。
-  `pack` 方法没有原因的返回（调用时应该重新检查导致线程暂停的条件）

```java
public static void park(Object blocker) {
    Thread t = Thread.currentThread();
    // 通过 Unsafe 设置当前 Thread 的 blocker 属性
    setBlocker(t, blocker);
  	// 通过 Unsafe 挂起当前线程
    U.park(false, 0L);
  	// 挂起的线程被唤醒后，将 Thread 的 blocker 属性置空
    setBlocker(t, null);
}
```

####park 的多版本

| park(Object)            | 挂起当前线程，具体见上面pack的源码分析                       |
| :---------------------- | ------------------------------------------------------------ |
| parkNanos(Object, long) | 指定了一个挂起时间（相对于当前的时间），时间到后自动被唤醒；例如1000纳秒后自动唤醒 |
| parkUntil(Object, long) | 指定一个挂起时间（**绝对时间**），时间到后自动被唤醒；例如2018-02-12 21点整自动被唤醒 |
| park()                  | 和park(Object)相比少了挂起前为线程设置blocker、被唤醒后清理blocker的操作 |
| parkNanos(long)         | 和parkNanos(Object, long)类似，仅少了blocker相关的操作       |
| parkUntil(long)         | 和parkUntil(Object, long)类似，仅少了blocker相关的操作       |

#### unpark

设置线程许可为可用。

- 如果线程当前已经被 `pack` 挂起，那么这个线程将会被唤醒。
- 如果线程当前没有被挂起，那么下次调用 `pack` 不会挂起线程。

## 分析

#### 挂起与阻塞

挂起与阻塞主要的区别应该说是它们面向的对象不同。对线程来说， *LockSupport* 的 `park/unpark`更符合阻塞和唤醒的语义，他们以“线程”作为方法的参数，语义更清晰，使用起来也更方便。而 `wait/notify` 使“线程”的阻塞/唤醒对线程本身来说是被动的，要准确的控制哪个线程、什么时候阻塞/唤醒是很困难的，所以是要么随机唤醒一个线程（`notify`）要么唤醒所有的（`notifyAll`）。

`park` 和 `unpark` 方法不会出现 `Thread.suspend` 和 `Thread.resume` 的死锁问题。这是因为 **许可** 的存在，调用 `park` 的线程和另一个试图将其 `unpark` 的线程之间的将没有竞争关系。此外，如果线程被中断或者超时，则 `park` 将返回。

`park` 方法还可以在其他任何时间“毫无理由”地返回，因此通常必须在重新检查返回条件的循环里调用此方法。从这个意义上说，`park` 是“忙碌等待”的一种优化，它不会浪费这么多的时间进行自旋，但是必须将它与 `unpark` 配对使用才更高效。



#### Why are Thread.suspend and Thread.resume deprecated? 
`Thread.suspend` is inherently deadlock-prone`天生就容易死锁`. If the target thread holds a lock on the monitor protecting a critical system resource when it is **suspended**, no thread can access this resource until the target thread is **resumed**. If the thread that would resume the target thread attempts to lock this monitor prior to calling resume, deadlock results. Such deadlocks typically manifest themselves as "frozen" processes. 