This file is organized to make things a little easier to follow while reading than they might otherwise: First the main static declarations and utilities, then fields, then main public methods (with a few factorings of multiple public methods into internal ones), then sizing methods, trees, traversers, and bulk operations.

## sourcecode

### 常量

```java
/**
 * The largest possible table capacity.  This value must be
 * exactly 1<<30 to stay within Java array allocation and indexing
 * bounds for power of two table sizes, and is further required
 * because the top two bits of 32bit hash fields are used for
 * control purposes.
 */
private static final int MAXIMUM_CAPACITY = 1 << 30;// 最大容量
private static final int DEFAULT_CAPACITY = 16; // 默认容量
static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;// 最大的数组大小
/**
 * The load factor for this table. Overrides of this value in
 * constructors affect only the initial table capacity.  The
 * actual floating point value isn't normally used -- it is
 * simpler to use expressions such as {@code n - (n >>> 2)} for
 * the associated resizing threshold.
 */
// 默认的加载因子，重写这个值，只会在初始化表格的时候用到
private static final float LOAD_FACTOR = 0.75f;
static final int TREEIFY_THRESHOLD = 8;// 树化阀值
static final int UNTREEIFY_THRESHOLD = 6;//非树化阀值
static final int MIN_TREEIFY_CAPACITY = 64;//允许树化时的最小表容量，低于这个值只会resize

/**
 * Minimum number of rebinnings重新排列 per transfer step. Ranges are
 * subdivided范围被细分 to allow multiple resizer threads.  This value
 * serves as a lower bound下限 to avoid resizers encountering遭遇
 * excessive过度的 memory contention争用.  The value should be at least
 * DEFAULT_CAPACITY.
 */
private static final int MIN_TRANSFER_STRIDE = 16;//最小转移步数
/**
 * The number of bits used for generation stamp in sizeCtl.
 * Must be at least 6 for 32bit arrays.
 */
private static final int RESIZE_STAMP_BITS = 16;
/**
 * The maximum number of threads that can help resize.
 * Must fit in 32 - RESIZE_STAMP_BITS bits.
 */
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
/**
 * The bit shift for recording size stamp in sizeCtl.
 */
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
/*
 * Encodings for Node hash fields. See above for explanation.
 */
static final int MOVED     = -1; // hash for forwarding nodes  转发节点
static final int TREEBIN   = -2; // hash for roots of trees    跟节点
static final int RESERVED  = -3; // hash for transient reservations   临时保留
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash 掩码
```

### 核心属性

```java
/**
 * The array of bins. Lazily initialized upon first insertion.
 * Size is always a power of two. Accessed directly by iterators.
 */
transient volatile Node<K,V>[] table;
private transient volatile Node<K,V>[] nextTable;// resizing 时候的扩容之后的表
/**
 * Base counter value, used mainly when there is no contention,
 * but also as a fallback during table initialization
 * races. Updated via CAS.
 */
private transient volatile long baseCount;
/**
 * Table initialization and resizing control.  When negative, the
 * table is being initialized or resized: -1 for initialization,
 * else -(1 + the number of active resizing threads).  Otherwise,
 * when table is null, holds the initial table size to use upon
 * creation, or 0 for default. After initialization, holds the
 * next element count value upon which to resize the table.
 */
private transient volatile int sizeCtl;
/**
 * The next table index (plus one) to split while resizing.
 */
private transient volatile int transferIndex;
/**
 * Spinlock (locked via CAS) used when resizing and/or creating CounterCells.
 */
private transient volatile int cellsBusy;
/**
 * Table of counter cells. When non-null, size is a power of 2.
 */
private transient volatile CounterCell[] counterCells;

// views
private transient KeySetView<K,V> keySet;
private transient ValuesView<K,V> values;
private transient EntrySetView<K,V> entrySet;
```

### 主要public方法

#### size 

```java
public int size() {
  long n = sumCount();
  return ((n < 0L) ? 0 :
          (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
          (int)n);
}

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

size 是汇总所有 CounterCell 的 value 值得到的总大小。

#### get

```java
/**
 * Returns the value to which the specified key is mapped,
 * or {@code null} if this map contains no mapping for the key.
 *
 * <p>More formally, if this map contains a mapping from a key
 * {@code k} to a value {@code v} such that {@code key.equals(k)},
 * then this method returns {@code v}; otherwise it returns
 * {@code null}.  (There can be at most one such mapping.)
 *
 * @throws NullPointerException if the specified key is null
 */
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
      	// 如果头节点就是，直接返回头节点的val
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 如果头节点是转发节点，则进行查询转发(可能正在resize等操作，此时需要转发到新表查询）
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
      
      	// 如果头节点不是，且这一部分节点也并未被转发走，则查询后续节点
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

```java
// 算法同 hashtable 不过多了一个 HASH_BITS 掩码
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;//HASH_BITS:usable bits of normal node hash
}
```

##### Special Nodes

get 方法在发现节点是特殊节点的时候，调用特殊节点的find 方法

默认的 find 实现如下，但是实际上是被各个子类重写，具体如下

```java
/**
 * Virtualized support for map.get(); overridden in subclasses.
 */
Node<K,V> find(int h, Object k) {
    Node<K,V> e = this;
    if (k != null) {
        do {
            K ek;
            if (e.hash == h &&
                ((ek = e.key) == k || (ek != null && k.equals(ek))))
                return e;
        } while ((e = e.next) != null);
    }
    return null;
}
```

各个特殊节点 find 实现如下：

##### ForwardingNode

```java
/**
 * A node inserted at head of bins during transfer operations.
 */
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null);
        this.nextTable = tab;
    }

    Node<K,V> find(int h, Object k) {
        // loop to avoid arbitrarily deep recursion on forwarding nodes
        outer: for (Node<K,V>[] tab = nextTable;;) {
            Node<K,V> e; int n;
          	// 如果新表没有，返回null
            if (k == null || tab == null || (n = tab.length) == 0 ||
                (e = tabAt(tab, (n - 1) & h)) == null)
                return null;
            for (;;) {
                int eh; K ek;
              	// 找到节点
                if ((eh = e.hash) == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                if (eh < 0) {
                  	// 如果目标位置也是一个转发节点，则重定向到该节点再继续处理
                    if (e instanceof ForwardingNode) {
                        tab = ((ForwardingNode<K,V>)e).nextTable;
                        continue outer;
                    }
                    else
                      	// 其他特殊节点转发
                        return e.find(h, k);
                }
                if ((e = e.next) == null)
                    return null;
            }
        }
    }
}
```

##### TreeBin

```java
/**
 * Returns matching node or null if none. Tries to search
 * using tree comparisons from root, but continues linear
 * search when lock not available.
 */
final Node<K,V> find(int h, Object k) {
    if (k != null) {
        for (Node<K,V> e = first; e != null; ) {
            int s; K ek;
            if (((s = lockState) & (WAITER|WRITER)) != 0) {
              	// 如果锁不可用的时候，采用 next 线性查找
                if (e.hash == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                e = e.next;
            }
            else if (U.compareAndSetInt(this, LOCKSTATE, s, s + READER)) { // 读锁 + 1
                TreeNode<K,V> r, p;
                try {
                  	// 红黑树查找
                    p = ((r = root) == null ? null :
                         r.findTreeNode(h, k, null));
                } finally {
                    Thread w;
                    if (U.getAndAddInt(this, LOCKSTATE, -READER) ==
                        (READER|WAITER) && (w = waiter) != null) // 读锁 -1 && 有等待者
                        LockSupport.unpark(w); // 唤醒等待线程
                }
                return p;
            }
        }
    }
    return null;
}
```

另外两个特殊的 Node 这里不再叙述，一个是 TreeNode, 原理是红黑树搜索，一个是空Node，永远返回NULL

#### put

```java
/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
  	// 不允许空 k v
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh; K fk; V fv;
        // 第一次 put 初始化 table
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { 
          	// 如果目标位置为空，则 CAS 放入
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED) // 如果目标位置标记为 MOVED
            tab = helpTransfer(tab, f);
	      // 就是头元素 && onlyIfAbsent && 目标位置已经有元素，不处理直接返回原值
        else if (onlyIfAbsent && fh == hash &&  // check first node
                 ((fk = f.key) == key || fk != null && key.equals(fk)) &&
                 (fv = f.val) != null)
            return fv; 
        else {
            V oldVal = null;
            synchronized (f) { // 锁 RootNode
                if (tabAt(tab, i) == f) { // 确定当前 node 没有被其他线程修改
                    if (fh >= 0) {// 非特殊节点 即链表
                        binCount = 1;// 当前节点的元素数量，用于判断是否需要树化
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                          	// 追加到尾部
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2; // 已经是树 则不需要扩容，这里直接写死为2
                      	// 树putTreeVal处理
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                  	// 转发节点，不做处理么？
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD) //树化
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

##### initTable

```java
/**
 * Initializes table, using the size recorded in sizeCtl.
 */
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) { 
          	// CAS 操作设置sc = -1，之后其他并发产生的冲突线程全部进入自旋
            try {
              	// 双重检查
                if ((tab = table) == null || tab.length == 0) {
                  	// sc 保留的是改之前的 sizeCtl，如果大于0，表示是initialCapacity
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2); // -1/4 即整个表容量的 3/4， threshold?
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

##### addCount

```java
/**
 * Adds to count, and if table is too small and not already
 * resizing, initiates transfer启动传输. If already resizing, helps
 * perform transfer if work is available.  Rechecks occupancy使用率
 * after a transfer to see if another resize is already needed
 * because resizings are lagging滞后 additions.
 *
 * @param x the count to add
 * @param check if <0, don't check resize, if <= 1 only check if uncontended
 */
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    if ((as = counterCells) != null/*第一次加*/ ||
        !U.compareAndSetLong(this, BASECOUNT, b = baseCount, s = b + x)/*并发失败*/) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSetLong(a, CELLVALUE, v = a.value, v + x))) {
          	// 需要初始化或者加失败 
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
  
  	// check && resize
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) { //如果小于0说明已经有线程在扩容操作
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1))
                  	// 后续线程加入扩容
                    transfer(tab, nt);
            }
            else if (U.compareAndSetInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
              	// 第一个线程执行扩容
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

```java
// See LongAdder version for explanation
private final void fullAddCount(long x, boolean wasUncontended/*存在并发冲突*/) {
    int h;
/*每个线程都会通过ThreadLocalRandom.getProbe() & m寻址找到属于它的CounterCell，然后进行计数。ThreadLocalRandom是一个线程私有的伪随机数生成器，每个线程的probe都是不同的（这点基于ThreadLocalRandom的内部实现，它在内部维护了一个probeGenerator，这是一个类型为AtomicInteger的静态常量，每当初始化一个ThreadLocalRandom时probeGenerator都会先自增一个常量然后返回的整数即为当前线程的probe，probe变量被维护在Thread对象中），可以认为每个线程的probe就是它在CounterCell数组中的hash code。
  	*/
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();      // force initialization
        h = ThreadLocalRandom.getProbe();
        wasUncontended = true;
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        CounterCell[] as; CounterCell a; int n; long v;
        if ((as = counterCells) != null && (n = as.length) > 0) {
            if ((a = as[(n - 1) & h]) == null) {
                if (cellsBusy == 0) {            // Try to attach new Cell
                    CounterCell r = new CounterCell(x); // Optimistic create
                    if (cellsBusy == 0 &&
                        U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            else if (U.compareAndSetLong(a, CELLVALUE, v = a.value, v + x))
                break;
            else if (counterCells != as || n >= NCPU)
                collide = false;            // At max size or stale
            else if (!collide)
                collide = true;
            else if (cellsBusy == 0 &&
                     U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
                try {
                    if (counterCells == as) {// Expand table unless stale
                        CounterCell[] rs = new CounterCell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        counterCells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            h = ThreadLocalRandom.advanceProbe(h);
        }
        else if (cellsBusy == 0 && counterCells == as &&
                 U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
            boolean init = false;
            try {                           // Initialize table
                if (counterCells == as) {
                    CounterCell[] rs = new CounterCell[2];
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            if (init)
                break;
        }
        else if (U.compareAndSetLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          // Fall back on using base
    }
}
```

```java
/**
 * Moves and/or copies the nodes in each bin to new table. See
 * above for explanation.
 */
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep清理打扫 before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSetInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
              	// 处理完毕之后，table指向新表
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            if (U.compareAndSetInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        else if ((f = tabAt(tab, i)) == null)
          	// 如果当前位置为空，则添加一个转发节点，
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED)// 转发节点，表示已经被处理了
            advance = true; // already processed
        else {
            synchronized (f) {// 锁 RootNode
                if (tabAt(tab, i) == f) {// 双重检查，确认
                    Node<K,V> ln, hn;
                    if (fh >= 0) { // >=0表示是普通链表, 树的话，hash存的是标识位-2
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0) // 不需移动
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else // 需要移动
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd); // 原来旧表的位置添加一个转发节点
                        advance = true;
                    }
                    else if (f instanceof TreeBin) { // 树移动
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);// 原来旧表的位置添加一个转发节点
                        advance = true;
                    }
                }
            }
        }
    }
}
```

## Overview

**The primary design goal of this hash table is to maintain concurrent readability** (typically method get(), but also iterators and related methods) **while minimizing update contention**. Secondary goals are to **keep space consumption`消耗` about the same or better than java.util.HashMap**, and to support high initial insertion rates on an empty table by many threads. 

This map usually acts as a binned (bucketed`桶装的`) hash table. Each key-value mapping is held in a **Node**. Most nodes are instances of the basic **Node** class with hash, key, value, and next fields. However, various subclasses exist: **TreeNodes** are arranged in balanced trees, not lists. <u>TreeBins hold the roots of sets of TreeNodes. ForwardingNodes are placed at`放置在` the heads of bins during resizing. ReservationNodes are used as placeholders while establishing values in computeIfAbsent and related methods.</u> The types TreeBin, ForwardingNode, and ReservationNode do not hold normal user keys, values, or hashes, and are readily distinguishable`可以区别的` during search etc because they have negative`负的` hash fields and null key and value fields. (These special nodes are either`要么` uncommon`不常见的` or transient`短暂的`, so the impact of carrying around some unused fields is insignificant`微不足道`.) 

**The table is lazily initialized to a power-of-two size upon the first insertion**. Each bin in the table normally contains a list of Nodes (most often, the list has only zero or one Node). Table accesses require volatile/atomic reads, writes, and CASes. Because there is no other way to arrange this without adding further indirections, we use intrinsics (jdk.internal.misc.Unsafe) operations. 

**We use the top (sign`符号位`) bit of Node hash fields for control purposes** -- it is available anyway because of addressing constraints`寻址限制`. **Nodes with negative hash fields are specially handled`特殊处理` or ignored in map methods.**

 **Insertion (via put or its variants`变种`) of the first node in an empty bin is performed by just CASing it to the bin**. This is by far`显然的` the most common case`最常见的情况` for put operations under most key/hash distributions. Other **update operations (insert, delete, and replace) require locks**. **<u>We do not want to waste the space required to associate a distinct lock object with each bin`我们不想浪费将不同的锁对象与每个bin关联所需的空间`, so instead use the first node of a bin list itself as a lock. Locking support for these locks relies on builtin`内置的` "synchronized" monitors.</u>** 

Using the first node of a list as a lock does not by itself suffice though`但这本身还不够`: **When a node is locked, any update must first validate that it is still the first node after locking it, and retry if not**. ·*Because new nodes are always appended to lists, once a node is first in a bin, it remains first until deleted or the bin becomes invalidated (upon resizing).* 

**The main disadvantage of per-bin locks is that other update operations on other nodes in a bin list protected by the same lock can stall`可能会暂停`,** for example when user equals() or mapping functions take a long time. However, statistically`统计地`, under random hash codes, this is not a common problem. Ideally`理想地`, the frequency of nodes in bins follows a Poisson distribution (http://en.wikipedia.org/wiki/Poisson_distribution) with a parameter of about 0.5 on average, given the resizing threshold of 0.75, although with a large variance because of resizing granularity. Ignoring variance, the expected occurrences of list size k are (exp(-0.5) * pow(0.5, k) / factorial(k)). The first values are: 

    0: 0.60653066 
    1: 0.30326533 
    2: 0.07581633 
    3: 0.01263606 
    4: 0.00157952 
    5: 0.00015795 
    6: 0.00001316 
    7: 0.00000094 
    8: 0.00000006 

more: less than 1 in ten million 

Lock contention probability for two threads accessing distinct elements is roughly `1 / (8 * #elements)` under random hashes. 

Actual实际中 hash code distributions分布 encountered遇到 in practice sometimes deviate脱离 significantly明显 from uniform randomness一致随机性. This includes the case when **N > (1<<30)**, so some keys MUST collide碰撞. Similarly for dumb愚蠢 or hostile敌意 usages in which multiple keys are designed to have identical完全相同 hash codes or ones that differs only in masked-out high bits只在掩码之外的高位上不同. **So we use a secondary strategy that applies when the number of nodes in a bin exceeds a threshold. These TreeBins use a balanced tree to hold nodes** (**<u>a specialized form of red-black trees</u>**), bounding search time to **O(log N).** Each search step in a TreeBin is at least twice as slow as in a regular list, but given that N cannot exceed (1<<64) (before running out of addresses) this bounds search steps, lock hold times, etc, to reasonable constants (roughly 100 nodes inspected per operation worst case) so long as keys are Comparable (which is very common -- String, Long, etc). TreeBin nodes (TreeNodes) also maintain the same "next" traversal pointers as regular nodes, so can be traversed in iterators in the same way. `TreeBin中的每个搜索步骤的速度至少是常规列表中的两倍，但是如果n 不超过  (1<<64)  (在地址用完之前)，只要键是可比较的(比如非常常见的：String、Long等等)，那么这个范围内的搜索步骤、锁保持时间等将限制在合理的常量(每个操作最坏的情况下大约检查100个节点)。TreeBin节点(TreeNodes)也与常规节点保持相同的“next”遍历指针，因此可以以相同的方式在迭代器中遍历。`

The table is **resized** when occupancy使用率 exceeds a percentage threshold (nominally, 0.75, but see below下文). Any thread noticing an overfull bin may assist in resizing after the initiating thread allocates and sets up the replacement array`在启动线程分配和设置替换数组后，任何发现过满bin的线程都可以帮助调整大小`. However, rather than stalling停止, these other threads may proceed with insertions etc. The use of TreeBins shields保护 us from the worst case effects of overfilling过度填充 while resizes are in progress使用TreeBins可以在调整大小的过程中保护我们免受过度填充的最坏情况影响. **Resizing proceeds by transferring bins, one by one, from the table to the next table**. However, threads claim small blocks of indices to transfer (via field `transferIndex`) before doing so, reducing contention``线程在这样做之前申请小的索引块(通过字段transferindex),从而减少争用.`` A generation `stamp` in field `sizeCtl` ensures that **resizings do not overlap**. **Because we are using power-of-two expansion, the elements from each bin must either stay at same index, or move with a power of two offset**. We eliminate`消除` unnecessary node creation by catching cases where old nodes can be reused because their `next` fields won't change. <u>On average, only about one-sixth of them need cloning when a table doubles</u>. The nodes they replace will be garbage collectable as soon as they are no longer referenced by any reader thread that may be in the midst of concurrently traversing table. Upon transfer, the old table bin contains only a special forwarding node (with hash field "MOVED") that contains the next table as its key. On encountering a forwarding node, access and update operations restart, using the new table. 

**Each bin transfer requires its bin lock, which can stall waiting for locks while resizing**. However, because other threads can join in and help resize rather than contend for locks, average aggregate waits become shorter as resizing progresses`但是，由于其他线程可以加入并帮助调整大小，而不是争夺锁，因此随着调整大小的进行，平均聚合等待时间会变短`. The transfer operation must also ensure that all accessible bins in both the old and new table are usable by any traversal`遍历`. This is arranged`被安排` in part`部分地` by proceeding from the last bin (table.length - 1) up towards the first. **Upon seeing a <u>forwarding node</u>, traversals (see class Traverser) arrange to move to the new table without revisiting nodes. To ensure that no intervening中间 nodes are skipped even when moved out of order, a <u>stack</u> (see class TableStack) is created on first encounter遇到 of a <u>forwarding node</u> during a traversal, to maintain its place if later processing the current table.** The need for these save/restore mechanics is relatively rare相对较少, but when one forwarding node is encountered, typically many more will be通常会出现更多. **So Traversers use <u>a simple caching scheme</u> to avoid creating so many new TableStack nodes.** 

The traversal scheme also applies to partial traversals of ranges of bins (via an alternate可选的 Traverser constructor) to support partitioned分区 aggregate聚合 operations. Also, read-only operations give up if ever forwarded to a null table, which provides support for shutdown-style clearing, which is also not currently implemented. 

Lazy table initialization minimizes最小化 footprint占用空间 until first use, and also avoids resizings when the first operation is from a `putAll`, constructor with map argument, or deserialization. These cases attempt to <u>override the initial capacity settings</u>, but harmlessly fail to take effect in cases of races. 

The **element count** is maintained using a specialization`定制` of `LongAdder`. We need to incorporate`合并` a specialization rather than just use a `LongAdder` in order to access implicit contention-sensing`竞争感知` that leads to creation of multiple `CounterCells`. The counter mechanics avoid contention on updates but can encounter`导致` cache thrashing`缓存震荡` if read too frequently during concurrent access. To avoid reading so often, resizing under contention is attempted only upon adding to a bin already holding two or more nodes. Under uniform hash distributions, the probability of this occurring at threshold is around **13%, meaning that only about 1 in 8 puts check threshold** (and after resizing, many fewer do so). 

**TreeBins use <u>a special form of comparison</u>`比较` for search and related operations** (which is the main reason we cannot use existing collections such as TreeMaps). TreeBins contain Comparable elements, but may contain others, as well as以及 elements that are Comparable but not necessarily Comparable for the same T, so we cannot invoke compareTo among them. To handle this, **the tree is ordered primarily by hash value, then by Comparable.compareTo order if applicable**适合. ***On lookup at a node, if elements are not comparable or compare as 0 then both left and right children may need to be searched in the case of tied hash values. (This corresponds to相当于 the full list search that would be necessary if all elements were non-Comparable and had tied hashes.)*** On insertion, to keep a total ordering (or as close as is required here) across rebalancings, we compare `classes` and `identityHashCodes` as tie-breakers`为了在重新平衡过程中保持总顺序（或尽可能接近此处的要求），我们将classes和identityHashCodes作为连接断开器进行比较.` The red-black balancing code is updated from pre-jdk-collections (http://gee.cs.oswego.edu/dl/classes/collections/RBCell.java) based in turn on Cormen, Leiserson, and Rivest "Introduction to Algorithms" (CLR). 

**TreeBins also require an additional locking mechanism. While list traversal is always possible by readers even during updates, tree traversal is not, mainly because of tree-rotations旋转 that may change the root node and/or its linkages`即使在更新过程中，读卡器也始终可以进行列表遍历，但树遍历并不是这样，主要是因为树的旋转可能会更改根节点和/或其链接.` TreeBins include a simple read-write lock mechanism parasitic on依赖于 the main bin-synchronization strategy: Structural adjustments associated with an insertion or removal are already bin-locked (and so cannot conflict with other writers) but must wait for ongoing`正在执行的` readers to finish. Since`由于` there can be only one`只能有一个` such waiter, we use a simple scheme`方案` using a single "`waiter`" field to block writers. However, readers need never block. If the root lock is held, they proceed along the slow traversal path (via next-pointers) until the lock becomes available or the list is exhausted, whichever无论谁 comes first. These cases are not fast, but maximize aggregate总体上的 expected throughput. **

Maintaining维护 API and serialization compatibility兼容 with previous versions of this class introduces several oddities古怪现象. Mainly: We leave untouched but unused constructor arguments referring to concurrencyLevel. We accept a `loadFactor` constructor argument, but apply it only to initial table capacity (which is the only time that we can guarantee to honor it.) We also declare an unused "Segment" class that is instantiated in minimal form only when serializing. 

Also, solely for仅仅为了 compatibility兼容 with previous versions of this class, it extends `AbstractMap`, even though all of its methods are overridden, so it is just useless baggage无用的包袱. 

This file is organized to make things a little easier to follow while reading than they might otherwise: First the main static declarations and utilities, then fields, then main public methods (with a few factorings of multiple public methods into internal ones), then sizing methods, trees, traversers, and bulk operations.



## javadoc

A hash table supporting **full concurrency** of retrievals and **high expected concurrency** for **updates**. This class obeys the same functional specification as `Hashtable`, and includes versions of methods corresponding to`相应于` each method of `Hashtable`. However, even though all operations are thread-safe, retrieval operations do not entail`必要，牵涉` locking, and there is not any support for locking the entire table in a way that prevents all access. This class is fully interoperable`可互相操作` with `Hashtable` in programs that rely on its thread safety but not on its synchronization details.

Retrieval`检索` operations (including `get`) generally do not block, so may overlap`重叠` with update operations (including `put` and `remove`). Retrievals reflect the results of the most recently completed update operations holding upon their onset`开端`. (More formally`更正式地`, **an update operation for a given key bears a happens-before relation with any (non-null) retrieval for that key reporting the updated value.**) For aggregate`聚合` operations such as `putAll` and `clear`, concurrent retrievals may reflect insertion or removal of only some entries. **Similarly, Iterators, Spliterators and Enumerations return elements reflecting the state of the hash table at some point at or since the creation of the iterator/enumeration. They do not throw `ConcurrentModificationException`. However, iterators are designed to be used by only one thread at a time**. Bear in mind`牢记` that <u>the results of aggregate`聚合` status methods including `size`, `isEmpty`, and `containsValue` are typically useful only when a map is not undergoing concurrent updates in other threads</u>. Otherwise the results of these methods reflect transient states`瞬时状态` that may be adequate for`适用于` monitoring or estimation`预估` purposes, but not for program control.

The table is dynamically expanded`动态扩展` when there are too many collisions`冲突` (i.e., keys that have distinct hash codes but fall into the same **slot** modulo the table size), with the expected average effect of maintaining roughly`大概` two bins per mapping (corresponding to`对应于` a 0.75 load factor threshold for resizing). There may be much variance`幅度,差额` around this average as mappings are added and removed, but overall`但是总体而言`, this maintains a commonly accepted time/space tradeoff`权衡` for hash tables. However, resizing this or any other kind of hash table may be a relatively`相对地` slow operation. When possible, it is a good idea to provide a size estimate`预估量` as an optional `initialCapacity` constructor argument. An additional optional `loadFactor` constructor argument provides a further means of customizing initial table capacity by specifying the table density to be used in calculating the amount of space to allocate for the given number of elements. Also, for compatibility`兼容` with previous versions of this class, constructors may optionally specify an expected `concurrencyLevel` as an additional hint`附加提示` for internal sizing. Note that **using many keys with exactly the same `hashCode()` is a sure way to slow down performance of any hash table. To ameliorate`改善` impact, when keys are `Comparable`, this class may use comparison order among keys to help break ties.**

A Set projection of a `ConcurrentHashMap` may be created (using `newKeySet()` or `newKeySet(int)`), or viewed (using `keySet(Object)` when only keys are of interest, and the mapped values are (perhaps transiently) not used or all take the same mapping value.

A `ConcurrentHashMap` can be used as a scalable frequency`频繁地` map (a form of histogram`柱状图` or multiset`多集`) by using `java.util.concurrent.atomic.LongAdder` values and initializing via `computeIfAbsent`. For example, to add a **count** to a `ConcurrentHashMap<String,LongAdder> freqs`, you can use `freqs.computeIfAbsent(key, k -> new LongAdder()).increment()`;

This class and its views and iterators implement all of the optional methods of the `Map` and `Iterator` interfaces.

Like `Hashtable` but unlike `HashMap`, this class **does not allow null to be used as a key or value**.

`ConcurrentHashMaps` support a set of sequential and parallel bulk operations that, unlike most Stream methods, are designed to be safely, and often sensibly, applied even with maps that are being concurrently updated by other threads; for example, when computing a snapshot summary of the values in a shared registry. There are three kinds of operation, each with four forms, accepting functions with keys, values, entries, and (key, value) pairs as arguments and/or return values. Because the elements of a `ConcurrentHashMap` are not ordered in any particular way, and may be processed in different orders in different parallel executions, the correctness`正确性` of supplied functions should not depend on any ordering, or on any other objects or values that may transiently change while computation is in progress; and except for `forEach` actions, should ideally`合理地` be side-effect-free无副作用. Bulk operations on `Map.Entry` objects do not support method `setValue`.

* **forEach**: Performs a given action on each element. A variant form`变形` applies a given transformation on each element before performing the action.
* **search**: Returns the first available non-null result of applying a given function on each element; skipping further search when a result is found.
* **reduce**: Accumulates`聚集` each element. The supplied reduction function cannot rely on ordering (more formally, it should be both associative`结合的` and commutative`交换的`). There are five variants变种:
  * Plain reductions. `空聚合？`(There is not a form of this method for (key, value) function arguments since there is no corresponding return type.)
  * Mapped reductions that accumulate the results of a given function applied to each element.
  * Reductions to scalar doubles, longs, and ints, using a given basis value.

**These bulk operations accept a `parallelismThreshold` argument. Methods proceed sequentially if the current map size is estimated to be less than the given threshold**. Using a value of Long.MAX_VALUE suppresses`限制` all parallelism. Using a value of 1 results`产生...结果` in maximal parallelism by partitioning into enough subtasks to fully utilize`充分利用` the `ForkJoinPool.commonPool()` that is used for all parallel computations. Normally, you would initially choose one of these extreme`极端` values, and then measure performance of using in-between values that trade off overhead versus throughput`在开销和吞吐量之间权衡`.

The concurrency properties of bulk operations follow from those of `ConcurrentHashMap`: **Any non-null result returned from get(key) and related access methods bears a happens-before relation with the associated insertion or update.** The result of any bulk operation reflects the composition`成分` of these per-element relations (but is not necessarily atomic with respect to`对于` the map as a whole unless it is somehow known to be quiescent`静态`). Conversely`相反`, because keys and values in the map are never null, null serves as a reliable atomic indicator`指示器` of the current lack of`缺少` any result. To maintain`维护` this property, `null` serves as an implicit basis`隐藏属性` for all non-scalar reduction`没有操作到数据` operations. For the double, long, and int versions, the basis should be one that, when combined with any other value, returns that other value (more formally, it should be the identity element for the reduction). Most common reductions have these properties; for example, computing a sum with basis 0 or a minimum with basis MAX_VALUE.

**Search and transformation functions provided as arguments should similarly return null to indicate the lack of any result** (in which case it is not used). In the case of mapped reductions, this also enables transformations to serve as filters, returning null (or, in the case of primitive specializations, the identity basis) if the element should not be combined. You can create compound`混合的` transformations and filterings by composing them yourself under this "null means there is nothing there now" rule before using them in search or reduce operations.

Methods accepting and/or returning Entry arguments maintain key-value associations. They may be useful for example when finding the key for the greatest value. Note that "plain" Entry arguments can be supplied using new `AbstractMap.SimpleEntry(k,v)`.

Bulk operations may complete abruptly, throwing an exception encountered`遇到` in the application of a supplied function. Bear in mind`牢记` when handling such exceptions that other concurrently executing functions could also have thrown exceptions, or would have done so if the first exception had not occurred.

Speedups`加速` for parallel compared to sequential forms are common but not guaranteed. Parallel operations involving brief`简单的` functions on small maps may execute more slowly than sequential forms if the underlying work`基础工作` to parallelize the computation is more expensive than the computation itself. Similarly, parallelization may not lead to much actual parallelism if all processors are busy performing unrelated tasks.

**All arguments to all task methods must be non-null.**

This class is a member of the Java Collections Framework.