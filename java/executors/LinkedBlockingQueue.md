# javadoc

An optionally-bounded **blocking queue** based on **linked** nodes. This queue orders elements **FIFO** (first-in-first-out). The head of the queue is that element that has been on the queue the longest time. The tail of the queue is that element that has been on the queue the shortest time. <u>New elements are inserted at the tail of the queue, and the queue retrieval operations obtain elements at the head of the queue</u>. **Linked queues typically have higher throughput than array-based queues but less predictable可预测 performance in most concurrent applications.**

The optional **capacity** bound constructor argument serves as a way to prevent excessive queue expansion`防止队列过度扩展`. The capacity, if unspecified, is equal to `Integer.MAX_VALUE`. Linked nodes are dynamically created upon each insertion unless this would bring the queue above capacity.

This class and its iterator implement all of the optional methods of the Collection and Iterator interfaces.
This class is a member of the Java Collections Framework.

#algorithm

A variant of the "two lock queue" algorithm. The **putLock** gates entry to `put` (and `offer`), and has an associated **condition** for waiting puts. Similarly for the **takeLock**. The "`count`" field that they both rely on is maintained as an **atomic** to avoid needing to get both locks in most cases. Also, to minimize need for puts to get takeLock and vice-versa, cascading notifies`级联通知` are used. When a put notices that it has enabled at least one take, it signals taker. That taker in turn signals others if more items have been entered since the signal. And symmetrically`对称地` for takes signalling puts. Operations such as `remove(Object)` and `iterators` acquire both locks. 

Visibility between writers and readers is provided as follows: 

Whenever an element is enqueued, the **putLock** is acquired and count updated. A subsequent reader guarantees visibility to the enqueued Node by either acquiring the putLock (via fullyLock) or by acquiring the takeLock, and then reading n = count.get(); this gives visibility to the first n items. 

To implement weakly consistent iterators, it appears we need to keep all Nodes GC-reachable from a predecessor dequeued Node. That would cause two problems:

- allow a rogue`恶意` Iterator to cause unbounded memory retention 
- cause cross-generational`跨代` linking of old Nodes to new Nodes if a Node was tenured`终身的` while live, which generational GCs have a hard time dealing with, causing repeated major collections. 

However, only non-deleted Nodes need to be reachable from dequeued Nodes, and reachability does not necessarily have to be of the kind understood by the GC. **We use the trick of linking a Node that has just been dequeued to itself. Such a self-link implicitly means to advance to head.next**.

# SourceCode

### Fields

```java
/**
 * Linked list node class
 * 队列节点，可以看出，队列是单向链表
 */
static class Node<E> {
    E item;

    /**
     * One of:
     * - the real successor Node
     * - this Node, meaning the successor is head.next
     * - null, meaning there is no successor (this is the last node)
     */
    Node<E> next;

    Node(E x) { item = x; }
}

/** The capacity bound, or Integer.MAX_VALUE if none */
private final int capacity;

/** Current number of elements */
private final AtomicInteger count = new AtomicInteger();

/**
 * Head of linked list.
 * Invariant: head.item == null
 */
transient Node<E> head;

/**
 * Tail of linked list.
 * Invariant: last.next == null
 */
private transient Node<E> last;

/** Lock held by take, poll, etc */
private final ReentrantLock takeLock = new ReentrantLock();

/** Wait queue for waiting takes */
private final Condition notEmpty = takeLock.newCondition();

/** Lock held by put, offer, etc */
private final ReentrantLock putLock = new ReentrantLock();

/** Wait queue for waiting puts */
private final Condition notFull = putLock.newCondition();
```

### Constructor

```java
/**
 * Creates a {@code LinkedBlockingQueue} with the given (fixed) capacity.
 *
 * @param capacity the capacity of this queue
 * @throws IllegalArgumentException if {@code capacity} is not greater
 *         than zero
 */
public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    last = head = new Node<E>(null); // 初始化收尾节点
}
```

### 核心方法

#### put

```java
/**
 * Inserts the specified element at the tail of this queue, waiting if
 * necessary for space to become available.
 *
 * @throws InterruptedException {@inheritDoc}
 * @throws NullPointerException {@inheritDoc}
 */
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    // Note: convention in all put/take/etc is to preset local var
    // holding count negative to indicate failure unless set.
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        /*
         * Note that count is used in wait guard even though it is
         * not protected by lock. This works because count can
         * only decrease at this point (all other puts are shut
         * out by lock), and we (or some other waiting put) are
         * signalled if it ever changes from capacity. Similarly
         * for all other uses of count in other wait guards.
         */
        while (count.get() == capacity) {
            notFull.await();
        }
        enqueue(node);
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            notFull.signal();// 注意这里是 signal 不是 signalAll
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
}
```

```java
/**
 * Links node at end of queue.
 *
 * @param node the node
 */
private void enqueue(Node<E> node) {
    // assert putLock.isHeldByCurrentThread();
    // assert last.next == null;
    last = last.next = node;
}
```

#### take

```java
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        while (count.get() == 0) {
            notEmpty.await();
        }
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();// 唤醒一个其他处于 take 等待的线程
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();// 唤醒一个处于 put 等待的线程
    return x;
}
```