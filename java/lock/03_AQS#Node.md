# AQS#Node

## 介绍

Node 描述了 AQS 锁中的一个节点信息。

AQS 锁中，Node 既表示 AQS waiterQueue 队列中的等待节点（此时，node 使用了 prev 与next 两个属性，组成了一个双向链表），也表示 ConditionObject  里面 ConditionQueue 等待队列中的 node节点（此时，node 使用了 nextWaiter 属性，组成了一个单向链表）。

## javadoc

Wait queue node class.
The wait queue is a variant`变体` of a "**CLH**" (Craig, Landin, and Hagersten) lock queue. CLH locks are normally used for **spinlocks**`自旋锁`. We instead use them for blocking synchronizers, but use the same basic tactic`策略` of holding some of the control information about a thread in the predecessor`前任` of its node. A "**status**" field in each node keeps track of whether a thread should block. A node is ***signalled*** when its predecessor releases. Each node of the queue otherwise serves as a specific-notification-style monitor holding a single waiting thread. The status field does NOT control whether threads are granted locks etc though. A thread may try to `acquire` if it is first in the queue. **But being first does not guarantee success; it only gives the right to contend.** So the currently released contender`竞争者` thread may need to rewait.
To enqueue`入队` into a CLH lock, you atomically splice`粘接` it in as new tail. To dequeue, you just set the head field.

```
			+------+  prev +-----+       +-----+
 head |      | <---- |     | <---- |     |  tail
			+------+       +-----+       +-----+
```

Insertion into a CLH queue requires only a single atomic operation on "tail", so there is a simple atomic point of demarcation`界限` from unqueued to queued. Similarly, dequeuing involves only updating the "head". However, it takes a bit more work for nodes to determine who their successors are, in part to`部分是为了` deal with possible cancellation due to timeouts and interrupts.
**The "*prev*" links (not used in original CLH locks), are mainly needed to handle *cancellation*. If a node is cancelled, its successor is (normally) <u>relinked</u> to a non-cancelled predecessor.** For explanation of similar mechanics in the case of spin locks, see the papers by Scott and Scherer at http://www.cs.rochester.edu/u/scott/synchronization/
We also use "**next**" links to implement blocking mechanics. The thread id for each node is kept in its own node, so a predecessor signals the next node to wake up by traversing`检索` next link to determine which thread it is. **Determination of successor must avoid races with newly queued nodes to set the "next" fields of their predecessors. This is solved when necessary by checking backwards from the atomically updated "tail" when a node's successor appears to be null. (Or, said differently, the next-links are an optimization so that we don't usually need a backward scan.)**

Cancellation introduces`引入` some conservatism`保守性` to the basic algorithms. Since`因为` we must poll`轮询` for cancellation of other nodes, we can miss noticing whether a cancelled node is ahead or behind us. This is dealt with`处理` by always *unparking* successors upon cancellation, allowing them to stabilize on a new predecessor, unless we can identify an uncancelled predecessor who will carry this responsibility.

**CLH** queues need a dummy`假的` header node to get started. But we don't create them on construction, because it would be wasted effort`精力` if there is never contention`争抢`. Instead, the node is constructed and head and tail pointers are set upon first contention.

**Threads waiting on Conditions use the same nodes, but use an additional`额外的` link. Conditions only need to link nodes in simple (non-concurrent) linked queues because they are only accessed when exclusively held. Upon await, a node is inserted into a condition queue. Upon signal, the node is transferred to the main queue. A special value of status field is used to mark which queue a node is on.**

```java
static final class Node {
    /** Marker to indicate a node is waiting in shared mode */
    static final Node SHARED = new Node();
    /** Marker to indicate a node is waiting in exclusive mode */
    static final Node EXCLUSIVE = null;

    /** waitStatus value to indicate thread has cancelled. */
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking. */
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition. */
    static final int CONDITION = -2;
    /**
     * waitStatus value to indicate the next acquireShared should
     * unconditionally propagate.
     */
    static final int PROPAGATE = -3;

    /**
     * Status field, taking on only the values:
     *   SIGNAL:     The successor of this node is (or will soon be)
     *               blocked (via park), so the current node must
     *               unpark its successor when it releases or
     *               cancels. To avoid races, acquire methods must
     *               first indicate they need a signal,
     *               then retry the atomic acquire, and then,
     *               on failure, block.
     *   CANCELLED:  This node is cancelled due to timeout or interrupt.
     *               Nodes never leave this state. In particular,
     *               a thread with cancelled node never again blocks.
     *   CONDITION:  This node is currently on a condition queue.
     *               It will not be used as a sync queue node
     *               until transferred, at which time the status
     *               will be set to 0. (Use of this value here has
     *               nothing to do with the other uses of the
     *               field, but simplifies mechanics.)
     *   PROPAGATE:  A releaseShared should be propagated to other
     *               nodes. This is set (for head node only) in
     *               doReleaseShared to ensure propagation
     *               continues, even if other operations have
     *               since intervened.
     *   0:          None of the above
     *
     * The values are arranged numerically to simplify use.
     * Non-negative values mean that a node doesn't need to
     * signal. So, most code doesn't need to check for particular
     * values, just for sign.
     *
     * The field is initialized to 0 for normal sync nodes, and
     * CONDITION for condition nodes.  It is modified using CAS
     * (or when possible, unconditional volatile writes).
     */
    volatile int waitStatus;

    /**
     * Link to predecessor node that current node/thread relies on
     * for checking waitStatus. Assigned during enqueuing, and nulled
     * out (for sake of GC) only upon dequeuing.  Also, upon
     * cancellation of a predecessor, we short-circuit while
     * finding a non-cancelled one, which will always exist
     * because the head node is never cancelled: A node becomes
     * head only as a result of successful acquire. A
     * cancelled thread never succeeds in acquiring, and a thread only
     * cancels itself, not any other node.
     */
    volatile Node prev;

    /**
     * Link to the successor node that the current node/thread
     * unparks upon release. Assigned during enqueuing, adjusted
     * when bypassing cancelled predecessors, and nulled out (for
     * sake of GC) when dequeued.  The enq operation does not
     * assign next field of a predecessor until after attachment,
     * so seeing a null next field does not necessarily mean that
     * node is at end of queue. However, if a next field appears
     * to be null, we can scan prev's from the tail to
     * double-check.  The next field of cancelled nodes is set to
     * point to the node itself instead of null, to make life
     * easier for isOnSyncQueue.
     */
    volatile Node next;

    /**
     * The thread that enqueued this node.  Initialized on
     * construction and nulled out after use.
     */
    volatile Thread thread;

    /**
     * Link to next node waiting on condition, or the special
     * value SHARED.  Because condition queues are accessed only
     * when holding in exclusive mode, we just need a simple
     * linked queue to hold nodes while they are waiting on
     * conditions. They are then transferred to the queue to
     * re-acquire. And because conditions can only be exclusive,
     * we save a field by using special value to indicate shared
     * mode.
     */
    Node nextWaiter;

    /**
     * Returns true if node is waiting in shared mode.
     */
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    /**
     * Returns previous node, or throws NullPointerException if null.
     * Use when predecessor cannot be null.  The null check could
     * be elided, but is present to help the VM.
     *
     * @return the predecessor of this node
     */
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    /** Establishes initial head or SHARED marker. */
    Node() {}

    /** Constructor used by addWaiter. */
    Node(Node nextWaiter) {
        this.nextWaiter = nextWaiter;
        THREAD.set(this, Thread.currentThread());
    }

    /** Constructor used by addConditionWaiter. */
    Node(int waitStatus) {
        WAITSTATUS.set(this, waitStatus);
        THREAD.set(this, Thread.currentThread());
    }

    /** CASes waitStatus field. */
    final boolean compareAndSetWaitStatus(int expect, int update) {
        return WAITSTATUS.compareAndSet(this, expect, update);
    }

    /** CASes next field. */
    final boolean compareAndSetNext(Node expect, Node update) {
        return NEXT.compareAndSet(this, expect, update);
    }

    final void setPrevRelaxed(Node p) {
        PREV.set(this, p);
    }

    // VarHandle mechanics
    private static final VarHandle NEXT;
    private static final VarHandle PREV;
    private static final VarHandle THREAD;
    private static final VarHandle WAITSTATUS;
    static {
        try {
            MethodHandles.Lookup l = MethodHandles.lookup();
            NEXT = l.findVarHandle(Node.class, "next", Node.class);
            PREV = l.findVarHandle(Node.class, "prev", Node.class);
            THREAD = l.findVarHandle(Node.class, "thread", Thread.class);
            WAITSTATUS = l.findVarHandle(Node.class, "waitStatus", int.class);
        } catch (ReflectiveOperationException e) {
            throw new Error(e);
        }
    }
}
```