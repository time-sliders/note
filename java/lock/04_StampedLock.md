# StampedLock

## 介绍

since jdk1.8

#### 适用场景

## sourcecode

## javadoc

**A capability-based`基于容量` lock** with **three modes** for controlling read/write access. The **state** of a `StampedLock` consists of a **version and mode**. Lock acquisition methods return a **stamp** that represents and controls access <u>with respect to</u>`关于` a lock state; "**try**" **versions of** these methods may instead return the special value *zero* to represent failure to acquire access. Lock release and conversion`转换` methods require stamps as arguments, and fail if they do not **match** the state of the lock. The three modes are:

* **Writing**. Method `writeLock` possibly blocks waiting for **exclusive** access, **returning a stamp that can be used in method `unlockWrite` to release the lock**. Untimed and timed versions of tryWriteLock are also provided. When the lock is held in write mode, no read locks may be obtained, and all optimistic`乐观` read validations will fail.
* **Reading**. Method `readLock` possibly blocks waiting for **non-exclusive** access, returning a stamp that can be used in method `unlockRead` to release the lock. Untimed and timed versions of tryReadLock are also provided.
* **Optimistic Reading**. Method `tryOptimisticRead` returns a non-zero **stamp** only if the lock is not currently held in write mode. **Method `validate` returns true if the lock has not been acquired in write mode since obtaining a given stamp.** *This mode can be thought of`认为是` as an extremely`非常` weak version of a read-lock, that can be broken by a writer at any time.* **The use of optimistic mode for short read-only code segments often <u>reduces contention</u> and <u>improves throughput</u>**. However, its use is inherently`内在` fragile`脆弱`. Optimistic read sections`部分` should only read fields and hold them in local variables <u>for later use after **validation**</u>. Fields read while in optimistic mode may be wildly`极度` inconsistent`不一致`, so usage applies only when you <u>are familiar enough with</u>`极度熟悉` data representations to check consistency and/or repeatedly invoke method `validate()`. For example, such steps are typically required when first reading an object or array reference, and then accessing one of its fields, elements or methods.

This class also supports methods that conditionally`有条件地` provide <u>conversions across</u>`转换` the three modes. For example, method `tryConvertToWriteLock` attempts to "upgrade" a mode, returning a valid write stamp if (1) already in writing mode (2) in reading mode and there are no other readers or (3) in optimistic mode and the lock is available. The forms of these methods are designed to help reduce some of the code bloat`臃肿` that otherwise`否则` occurs in retry-based`基于重试` designs.

StampedLocks are **designed for use as internal utilities`公共程序` in the development of thread-safe components**. Their use relies on knowledge of`了解` the internal properties of the data, objects, and methods they are protecting. They are **not reentrant**, so locked bodies should not call other unknown methods that may try to re-acquire locks (although you may pass a stamp to other methods that can use or convert it). The use of read lock modes relies on the associated code sections being side-effect-free`无副作用`. Unvalidated`未验证的` optimistic read sections cannot call methods that are not known to`不知道` <u>tolerate potential inconsistencies</u>`容忍潜在的不一致性`. Stamps use finite`有限的` representations, and are not cryptographically`密码学` secure`保护` (i.e., a valid stamp may be guessable`可猜测的`). Stamp values may **recycle** after (no sooner than) one year of continuous operation. A stamp held without use or validation for longer than this period may fail to validate correctly. StampedLocks are serializable, but always deserialize into initial unlocked state, so they are not useful for remote locking.

  Like `Semaphore`, but unlike most `Lock` implementations, StampedLocks **have no notion of ownership**. Locks acquired in one thread can be released or converted in another.
  **The scheduling policy of StampedLock does not consistently prefer`一贯喜欢` readers over writers or <u>vice versa</u>`反之亦然`**. All "try" methods are best-effort and do not necessarily conform to any scheduling or fairness policy. A zero return from any "try" method for acquiring or converting locks does not carry any information about the state of the lock; a subsequent`随后的` invocation may succeed.
  Because it supports **coordinated** usage across multiple lock modes, this class does not directly implement the `Lock` or `ReadWriteLock` interfaces. However, a `StampedLock` may be viewed `asReadLock()`, `asWriteLock()`, or `asReadWriteLock()` in applications requiring only the associated set of functionality.

  **Sample Usage**. The following illustrates`表明` some usage idioms`习惯` in a class that maintains simple two-dimensional`二维的` points. The sample code illustrates some try/catch conventions even though they are not strictly needed here because no exceptions can occur in their bodies.

```java
 class Point {
   private double x, y;
   private final StampedLock sl = new StampedLock();

   void move(double deltaX, double deltaY) { // an exclusively locked method
     long stamp = sl.writeLock();
     try {
       x += deltaX;
       y += deltaY;
     } finally {
       sl.unlockWrite(stamp);
     }
   }

   double distanceFromOrigin() { // A read-only method
     long stamp = sl.tryOptimisticRead();
     double currentX = x, currentY = y;
     if (!sl.validate(stamp)) {
       stamp = sl.readLock();
       try {
         currentX = x;
         currentY = y;
       } finally {
         sl.unlockRead(stamp);
       }
     }
     return Math.sqrt(currentX * currentX + currentY * currentY);
   }

   void moveIfAtOrigin(double newX, double newY) { // upgrade
     // Could instead start with optimistic, not read mode
     long stamp = sl.readLock();
     try {
       while (x == 0.0 && y == 0.0) {
         long ws = sl.tryConvertToWriteLock(stamp);
         if (ws != 0L) {
           stamp = ws;
           x = newX;
           y = newY;
           break;
         }
         else {
           sl.unlockRead(stamp);
           stamp = sl.writeLock();
         }
       }
     } finally {
       sl.unlock(stamp);
     }
   }
 }
```



## Algorithmic notes

Algorithmic notes: The design employs elements of Sequence locks (as used in linux kernels; see Lameter's http://www.lameter.com/gelato2005.pdf and elsewhere; see Boehm's http://www.hpl.hp.com/techreports/2012/HPL-2012-68.html) and Ordered RW locks (see Shirako et al http://dl.acm.org/citation.cfm?id=2312015) 

Conceptually·, the primary state of the lock includes a sequence number that is odd`奇数` when write-locked and even otherwise. However, this is **offset** by a reader count that is non-zero when read-locked. The read count is ignored when validating "optimistic" seqlock-reader-style stamps. Because we must use a small finite`有限的` number of bits (currently 7) for readers, **a supplementary`额外的` reader overflow word is used when the number of readers exceeds the count field**. We do this by treating the max reader count value (RBITS) as a spinlock protecting overflow updates. 

Waiters use a *modified* form of CLH lock used in `AbstractQueuedSynchronizer` (see its internal documentation for a fuller account), where each **node** **is tagged** (field **mode**) as either a reader or writer. **Sets of waiting readers are grouped** (linked) under a common node (field **cowait**) so act as a single node with respect to most CLH mechanics. By virtue of`有...优点` the queue structure, wait nodes need not actually carry sequence numbers; we know each is greater than its predecessor. This simplifies the scheduling policy to a mainly-FIFO scheme`方案` that incorporates`包含` elements of `Phase-Fair阶段公平` locks (see Brandenburg & Anderson, especially http://www.cs.unc.edu/~bbb/diss/). In particular, we use the phase-fair anti-barging`无冲突` rule: If an incoming reader arrives while read lock is held but there is a queued writer, this incoming reader is queued. (This rule is responsible for`负责` some of the complexity of method `acquireRead`, but without it`如果不这样`, the lock becomes highly unfair.) Method `release` does not (and sometimes cannot) itself wake up **cowaiters**. This is done by the primary thread, but helped by any other threads with nothing better to do in methods `acquireRead` and `acquireWrite`. 

These rules apply to threads actually queued. All `tryLock` forms opportunistically`机会主义的` try to acquire locks regardless of preference rules, and so may "barge" their way in. **Randomized spinning** is used in the `acquire` methods to **reduce (increasingly expensive`越来越贵`) context switching while also avoiding sustained`持续的` memory thrashing`颠簸,震荡` among many threads**. We limit spins to the head of queue. If, upon wakening, a thread fails to obtain lock, and is still (or becomes) the first waiting thread (which indicates that some other thread barged and obtained lock), it escalates`加剧` spins (up to MAX_HEAD_SPINS) to reduce the likelihood`可能` of continually losing to barging threads. 

Nearly all of these mechanics are carried out in methods `acquireWrite` and `acquireRead`, that, as typical of such code, sprawl out`散开` because actions and retries rely on consistent sets of locally cached reads. 

As noted in Boehm's paper (above), sequence validation (mainly method `validate()`) requires stricter ordering rules than apply to normal volatile reads (of "state"). To force orderings of reads before a validation and the validation itself in those cases where this is not already forced, we use `acquireFence`. Unlike in that paper, we allow writers to use plain writes. One would not expect reorderings of such writes with the lock acquisition CAS because there is a "control dependency", but it is theoretically possible, so we additionally add a `storeStoreFence` after lock acquisition CAS. 

----------------------------------------------------------------

Here's an informal proof`证明` that plain reads by _successful_ readers see plain writes from preceding but not following writers (following Boehm and the C++ standard [atomics.fences]): 

Because of the total synchronization order of accesses to volatile long state containing the sequence number, writers and _successful_ readers can be globally sequenced. 

```c++
int x, y; 

Writer 1: 
inc sequence (odd - "locked") 
storeStoreFence();
x = 1; y = 2; 
inc sequence (even - "unlocked") 

Successful Reader: 
read sequence (even) 
// must see writes from Writer 1 but not Writer 2 
r1 = x; r2 = y; 
acquireFence(); 
read sequence (even - validated unchanged) 
// use r1 and r2 

Writer 2: 
inc sequence (odd - "locked") 
storeStoreFence(); 
x = 3; y = 4; 
inc sequence (even - "unlocked") 
```

Visibility of writer 1's stores is normal - reader's initial read of state synchronizes with writer 1's final write to state. Lack of visibility of writer 2's plain writes is less obvious. If reader's read of x or y saw writer 2's write, then (assuming semantics of C++ fences) the storeStoreFence would "synchronize" with reader's acquireFence and reader's validation read must see writer 2's initial write to state and so validation must fail. But making this "proof" formal and rigorous is an open problem! 

----------------------------------------------------------------

The memory layout keeps lock state and queue pointers together (normally on the same cache line). This usually works well for read-mostly loads. In most other cases, the natural tendency of adaptive-spin CLH locks to reduce memory contention lessens motivation to further spread out contended locations, but might be subject to future improvements.