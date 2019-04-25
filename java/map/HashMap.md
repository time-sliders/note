## sourcecode

#### 基本属性介绍

##### 常量

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // 默认的初始化容量 16
static final int MAXIMUM_CAPACITY = 1 << 30; // 最大容量
static final float DEFAULT_LOAD_FACTOR = 0.75f; // 扩容因子
static final int TREEIFY_THRESHOLD = 8; // 树化阀值，当某一个 bin 达到这个值之后，会被树化
static final int UNTREEIFY_THRESHOLD = 6;// 拆树阀值，当某一个 bin 低于这个值之后，会被拆树
static final int MIN_TREEIFY_CAPACITY = 64; // 最小的树化容量，否则扩容
```

##### 属性

```java
/**
 * The table, initialized on first use, and resized as
 * necessary. When allocated, length is always a power of two.
 * (We also tolerate length zero in some operations to allow
 * bootstrapping mechanics that are currently not needed.)
 */
transient Node<K,V>[] table;
/**
 * Holds cached entrySet(). Note that AbstractMap fields are used
 * for keySet() and values().
 */
transient Set<Map.Entry<K,V>> entrySet;
transient int size;//The number of key-value mappings contained in this map.
/**
 * The number of times this HashMap has been structurally modified
 * Structural modifications are those that change the number of mappings in
 * the HashMap or otherwise modify its internal structure (e.g.,
 * rehash).  This field is used to make iterators on Collection-views of
 * the HashMap fail-fast.  (See ConcurrentModificationException).
 */
transient int modCount;
int threshold;// The next size value at which to resize (capacity * load factor).
final float loadFactor;//  The load factor for the hash table.
```

##### 数据类型

在树化之前，数组的每一个元素，都是一个单向 Node 链表

```java
/**
 * Basic hash bin node, used for most entries.  (See below for
 * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
 */
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
		.... 
}
```

在树化之后，节点变成一个红黑树的 TreeNode

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    ....
}
```

综上，可以看出hashmap的数据结构如下图：

// TODO

##### 构造方法

```java
/**
 * Constructs an empty {@code HashMap} with the specified initial
 * capacity and load factor.
 *
 * @param  initialCapacity the initial capacity
 * @param  loadFactor      the load factor
 * @throws IllegalArgumentException if the initial capacity is negative
 *         or the load factor is nonpositive
 */
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}
```

这里只是基本验证，需要注意 `tableSizeFor` 方法

```java
/**
 * Returns a power of two size for the given target capacity.
 */
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

该方法的作用是返回：**大于或等于输入参数且最近的2的整数次幂的数**

对n右移1位：`001xx...xxx`，再位或：`011xx...xxx`  

对n右移2为：`00011...xxx`，再位或：`01111...xxx`

此时前面已经有四个1了，再右移4位且位或可得8个1

同理，有8个1，右移8位肯定会让后八位也为1。

综上可得，该算法**让最高位的1后面的位全变为1**

**最后再让结果n+1，即得到了2的整数次幂的值了**

现在回来看第一条语句：

```java
int n = cap - 1;
```

让 `cap-1` 再赋值给 n 的目的是另找到的目标值大于或 **等于** 原值。例如二进制 1000，十进制数值为 8。如果不对它减1而直接操作，将得到答案 10000，即 16。显然不是结果。减1后二进制为111，再进行操作则会得到原来的数值 1000，即 8。

##### 核心方法

######put

```java
/**
 * Associates the specified value with the specified key in this map.
 * If the map previously contained a mapping for the key, the old
 * value is replaced.
 *
 * @return the previous value associated with {@code key}, or
 *         {@code null} if there was no mapping for {@code key}.
 *         (A {@code null} return can also indicate that the map
 *         previously associated {@code null} with {@code key}.)
 */
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

这里重点关注下 hash 与 putVal 函数，先看 hash 函数的实现

```java
/**
 * Computes key.hashCode() and spreads扩展 (XORs异或) higher bits of hash
 * to lower.  Because the table uses power-of-two masking二位掩码, sets of
 * hashes that vary不同 only in bits above the在...上面 current mask will
 * always collide. (Among known examples are sets of Float keys
 * holding consecutive whole numbers in small tables.)  So we
 * apply a transform that spreads the impact影响 of higher bits
 * downward向下. There is a tradeoff权衡 between speed, utility, and
 * quality of bit-spreading. Because many common sets of hashes
 * are already reasonably合理的 distributed (so don't benefit from
 * spreading), and because we use trees to handle large sets of
 * collisions in bins, we just XOR some shifted bits in the
 * cheapest possible way尽可能低的方式 to reduce systematic lossage有规律的丢失, 
 * as well as to incorporate合并 impact of the highest bits that would otherwise
 * never be used in index calculations because of table bounds.
 */
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

这段代码叫 **扰动函数**

理论上散列值是一个int型，如果直接拿散列值作为下标访问 HashMap 主数组的话，考虑到2进制32位带符号的int表值范围从**-2147483648**到**2147483648**。前后加起来大概40亿的映射空间。只要哈希函数映射得比较均匀松散，一般应用是很难出现碰撞的。

但问题是一个40亿长度的数组，内存是放不下的。HashMap扩容之前的数组初始大小才16。所以这个散列值是不能直接拿来用的。用之前还要先做对数组的长度取模运算，得到的余数才能用来访问数组下标。源码中模运算是在这个indexFor( )函数里完成的。

```java
static int indexFor(int h, int length/*数组长度*/) {
    return h & (length-1);
}
```

indexFor 的代码也很简单，就是把散列值和数组长度做一个"与"操作，顺便说一下，这也正好解释了为什么HashMap的数组长度要取2的整数幂。因为这样（数组长度-1）正好相当于一个**低位掩码**。“与”操作的结果就是散列值的高位全部归零，只保留低位值，用来做数组下标访问。以初始长度16为例，16-1=15。2进制表示是00000000 00000000 00001111。和某散列值做“与”操作如下，结果就是截取了最低的四位值。

```java
    10100101 11000100 00100101
&   00000000 00000000 00001111
----------------------------------
    00000000 00000000 00000101    //高位全部归零，只保留末四位
```

但这时候问题就来了，这样就算我的散列值分布再松散，要是**只取最后几位的话，碰撞会很严重**。更要命的是如果散列本身做得不好，分布上成等差数列的漏洞，恰好使最后几个低位呈现规律性重复，就无比蛋疼。

时候“**扰动函数**”的价值就体现出来了，说到这里大家应该猜出来了。看下面这个图

![hash](ref/hash.jpg)

右位移16位，正好是32bit的一半，自己的高半区和低半区做异或，就是为了**混合原始哈希码的高位和低位，以此来加大低位的随机性**。而且混合后的低位掺杂了高位的部分特征，这样**高位的信息也被变相保留**下来。

###### putVal

```java
/**
 * Implements Map.put and related methods
 *
 * @param hash hash for key
 * @param key the key
 * @param value the value to put
 * @param onlyIfAbsent if true, don't change existing value
 * @param evict if false, the table is in creation mode.
 * @return previous value, or null if none
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 如果 node 数组尚未创建，则调用 resize() 初始化数组
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
  
  	// (n - 1) & hash 是计算 hash 在数组的哪个 node 存储，这里如果不存在，则初始化node
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
  
  	// 如果 node 已经存在，则需要考虑怎么追加到node后面，根据 node 的数据类型，这里分为很多情况
    else {
      	/*
      	 * 1. 找位置
      	 */
        Node<K,V> e/* 要放入的位置 */; K k;
      	
      	// 如果存入的 hash == root-node 的 hash 或者 key 相同，则记录要放入在 root-node
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
      
      	// 如果是树节点，则调用树节点的 putTreeVal
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
      
      	// 如果不是树节点，即表示是链表，则
        else {
            for (int binCount = 0; ; ++binCount) {/*bins就是容器的意思，这里指node数量*/
                if ((e = p.next) == null) {// 遍历到了队尾
                    p.next = newNode(hash, key, value, null);
                    // 如果当前链表大于树化阀值，则执行链表树化
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash); // 树化
                    break;
                }
                // 找到了合适的位置，则中断查找
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                // 指针后移，继续查找
                p = e;
            }
        }
      
      	/*
      	 * 2. 放数据
      	 */
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount; //如果发生了数据结构的变动，则++，fail-fast iterator
    if (++size > threshold) // size 大于 threshold，扩容
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

###### resize 

```java
/**
 * Initializes or doubles table size.  If null, allocates in
 * accord with initial capacity target held in field threshold.
 * Otherwise, because we are using power-of-two expansion, the
 * elements from each bin must either stay at same index, or move
 * with a power of two offset in the new table.
 *
 * @return the table
 */
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
  	/**
  	 * 计算新的容量与扩容阀值
  	 */
    if (oldCap > 0) {
        // 不允许超过最大容量
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 容量 * 2 && 扩容阀值 * 2
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr; // 初始容量 == 初始阀值
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
      	// 初始化扩容阀值
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
  
  	/**
  	 * 根据新的 cap 与 thr 重构数组
  	 */
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) { // 依次将旧的元素放入新的数组
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
              
              	/** 如果当前node 没有后续节点，根据hash算法，新的位置肯定也是空的
              	 * 原因，每次扩容，cap << 1 移动一位，对于一个固定的 hash 值
              	 * 如果在多出来的这一位上是 1，则会根据hash 算法，计算出一个新的位置
              	 * 并移过去，如果是0，则不需要移动
              	 * 所以这里，如果没有 next，是可以直接移动的
              	 */
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
              
              	// 如果是 tree node，则需要对树进行拆分，主要是hash值多出来的掩码是 1的拆出去
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
              
                else { // preserve保留 order
                    Node<K,V> loHead = null, loTail = null; // 掩码位是 0 的：保留
                    Node<K,V> hiHead = null, hiTail = null; // 掩码位是 1 的：移走
                    Node<K,V> next;
                    do {
                        next = e.next;
                      	// 根据掩码位 0 / 1，将链表拆分为需要移走和需要保留的2个链表
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                  
                    if (loTail != null) {
                        loTail.next = null;
                  			// 保留的部分添加到新数组原来的老的位子
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                      	// 移走的部分添加新数组到新对应的新的位置
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

从该方法可以看到，数组是在第一次放入数据的时候初始化的，如果外部传入了一个 initcap 则初始化容量是 initcap 的下一个2的整数次幂，thr = initcap * loadFactor; 比如传入12 ，实际数组大小为 cap = 16 ，thr = 12. 从而保证塞满12个元素不需要扩容。

**resize 核心逻辑：每次扩容，cap 向左移动一位，对于 HashMap#hash 算法来说，相当于，在原来的 length-1 掩码的高位，多出来了一位掩码。对于一个任意key 的 hashcode 来计算的话，这一位可能是1，也可能是0，如果是1，则这部分数据，需要移动到新的位置，如果是0，则需要保留在原来的位置。**

###### treeifyBin

```java
/**
 * Replaces all linked nodes in bin at index for given hash unless
 * table is too small, in which case resizes instead.
 */
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
  	
  	// 如果表格很小，只进行扩容，不树化
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
 		
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd/*树头*/ = null, tl/*树尾*/ = null;
      	/**
      	 * 1. 将 Node 转换为 TreeNode, 这里仅仅做数据转换，然后做成链表首位相连，还未排序
      	 */
        do {
          	// 将一个 Node 转换为 TreeNode，
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
      
      	/**
      	 * 2. 树化：将第一步的链表转换为树
      	 */
        if ((tab[index] = hd) != null)
            hd.treeify(tab);  // 树化核心代码
    }
}
```

###### remove

```java
/**
 * Implements Map.remove and related methods
 *
 * @param hash hash for key
 * @param key the key
 * @param value the value to match if matchValue, else ignored
 * @param matchValue if true only remove if value is equal
 * @param movable if false do not move other nodes while removing
 * @return the node, or null if none
 */
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
  	// 找到对应的根 Node
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
      	/*
      	 * 1. 找 Node
      	 */
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k)))) // 是 Root Node
            node = p; 
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                // 如果是 TreeNode，则从树中获取
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
              	// 如果是链表，则遍历链表查找
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
      
      	/*
      	 * 2. 删元素
      	 */
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
              	// 从树中删除
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
              	// 如果是删头节点，则吧头节点的下一个元素移到头节点。
                tab[index] = node.next;
            else
                p.next = node.next; // 删除当前节点
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

## Implementation notes

This map usually acts as a binned`装箱` (bucketed) hash table, but **when bins get too large, they are transformed into bins of TreeNodes**, each structured similarly to those in `java.util.TreeMap`. Most methods try to use normal bins, but relay`转发` to `TreeNode` methods when applicable`合适` (simply by checking instanceof a node). Bins of `TreeNodes` may be traversed and used like any others, but additionally support faster lookup when overpopulated`过于拥挤`. However, since the vast majority of`绝大多数` bins in normal use are not overpopulated, checking for existence of`存在性` tree bins may be delayed in the course of`在…的过程中` table methods. 

Tree bins (i.e., bins whose elements are all TreeNodes) are **ordered primarily by hashCode**, but in the case of ties`捆绑`, if two elements are of the same "class C implements Comparable ", type then their `compareTo` method is used for ordering. (We conservatively`保守地` check generic types`范形` via reflection to validate this -- see method `comparableClassFor`). The added complexity`增加的复杂性` of tree bins is worthwhile in providing worst-case **O(log n)** operations when keys either have distinct hashes or are orderable`在key具有不同的hashes或者可以排序时，为了保证在最坏情况下提供 O(logn)的操作复杂度，对tree bins 增加额外的复杂性是值得的`, Thus, performance degrades`降低` gracefully`优雅地` under accidental`意外的` or malicious`恶意的` usages in which hashCode() methods return values that are poorly`差` distributed`分布`, as well as`或者` those in which many keys share a hashCode, so long as`只要` they are also Comparable.`因此，在返回分布不好的hashcode以及许多键共享一个hashcode，等等意外或恶意使用hashcode()的情况下，只要它们是Comparable的，性能就能够保证优雅的降低` (If neither of these apply, we may waste about a factor of two in time and space compared to taking no precautions`如果这两种方法都不适用，与不采取预防措施相比，我们可能会浪费大约两倍的时间和空间`. But the only known cases stem`来自` from poor`糟糕的` user programming practices that are already so slow that this makes little difference`但是，唯一已知的情况来自于糟糕的用户编程实践，这些实践已经非常缓慢，以至于没有什么区别`.) 

Because TreeNodes are about twice the size of regular`常规` nodes, we use them only when bins contain enough nodes to warrant`保证` use (see *TREEIFY_THRESHOLD*). And **when they become too small (due to removal or resizing) they are converted back to plain bins**. In usages with well-distributed`分布良好` user hashCodes, tree bins are rarely`很少` used. Ideally`理想情况下`, under random hashCodes, the frequency of`频率` nodes in bins follows a Poisson distribution (http://en.wikipedia.org/wiki/Poisson_distribution) with a parameter of about 0.5 on average for the default resizing threshold of 0.75`对于默认的0.75大小调整阈值，平均参数约为0.5`, although`然而，但是` with a large variance`差异` because of resizing granularity`粒度`. Ignoring variance`方差`, the expected occurrences of list size k are `(exp(-0.5) * pow(0.5, k) / factorial(k))`. The first values are: 

0: 	0.60653066 
1:	 0.30326533 
2: 	0.07581633 
3: 	0.01263606 
4: 	0.00157952 
5: 	0.00015795 
6:	 0.00001316 
7:	 0.00000094 
8:	 0.00000006

more: less than 1 in ten million 

The root of a tree bin is normally its first node. However, sometimes (currently only upon Iterator.remove), the root might be elsewhere`其他位置`, but can be recovered`恢复` following parent links (method `TreeNode.root()`).

 All applicable`适用的` internal methods accept a hash code as an argument (as normally supplied from a public method), allowing them to call each other without recomputing user hashCodes. Most internal methods also accept a "**tab**" argument, that is normally **the current table**, but may be a new or old one when resizing or converting. 

When bin lists are treeified`树化的`, split, or untreeified, we **keep them in the same relative access/traversal order** (i.e., field Node.next) to better preserve locality`为了更好地保留位置`, and to slightly simplify`稍微减缓` handling of splits and traversals that invoke `iterator.remove`. When using comparators on insertion, to keep a total ordering (or as close as is required here`或者尽可能接近这里的要求`) across rebalancings`重平衡`, we compare classes and identityHashCodes as tie-breakers`断路器`. 

The use and transitions`转换` among plain`普通` vs tree modes is complicated`复杂` by the existence`存在` of subclass `LinkedHashMap`. See below for **hook`钩子` methods defined to be invoked upon insertion, removal and access that allow `LinkedHashMap` internals to otherwise remain independent`独立于` of these mechanics**. (This also requires that a map instance be passed to some utility methods that may create new nodes.`这还需要将map实例传递给一些可能会创建新节点的公用方法`) 

The concurrent-programming-like SSA-based coding style helps avoid aliasing errors`混叠错误` amid all of`在所有..` the twisty pointer`弯曲指针，非直接指针` operations.

## javadoc

Hash table based implementation of the `Map` interface. This implementation provides all of the optional map operations, and **permits `null` values and the `null` key**. (The `HashMap` class is roughly`大体上` equivalent to `Hashtable`, except that it is **unsynchronized and permits nulls**.) This class makes **no guarantees as to the order** of the map; in particular, it does not guarantee that the order will remain constant`保持不变` over time`随着时间的推移`.

This implementation provides constant-time performance`恒时性能` for the basic operations (`get` and `put`), assuming`假设` the hash function disperses`分散` the elements properly`适当的` among the buckets`桶`. Iteration over collection views requires time`需要的时间` proportional to`与…成正比` the "capacity" of the HashMap instance (the number of buckets) plus`加上` its size (the number of key-value mappings). Thus, it's very important **not to set the initial capacity too high** (**or the load factor too low**) if iteration performance is important.

An instance of `HashMap` has two parameters that affect its performance: **initial capacity** and **load factor**. The capacity is the number of buckets in the hash table, and the initial capacity is simply the capacity at the time the hash table is created. The load factor is a measure of`是衡量` how full the hash table is allowed to get before its capacity is automatically increased. When the number of entries in the hash table exceeds the product of`与…的乘积` the load factor and the current capacity, the hash table is **rehashed** (that is, **internal data structures are rebuilt**) so that the hash table has approximately`大约` twice the number of buckets.

As a general rule, the default load factor (.75) offers a good tradeoff`权衡，折中` between time and space costs. Higher values decrease the space overhead`开销` but increase the lookup`查询` cost (reflected in most of the operations of the `HashMap` class, including `get` and `put`). The expected number of entries in the map and its load factor should be taken into account`考虑到` when setting its initial capacity, so as to **minimize the number of rehash operations**. If the **initial capacity is greater than the maximum number of entries divided by the load factor**, no rehash operations will ever occur.

If many mappings are to be stored in a `HashMap` instance, creating it with a sufficiently`充分的` large capacity will allow the mappings to be stored more efficiently`效率地` than letting it perform automatic rehashing as needed to grow the table. Note that **using many keys with the same `hashCode()` is a sure way to slow down performance of any hash table**. To ameliorate`改善` impact`影响`, when keys are `Comparable`, this class may use comparison order among keys to help break ties`捆绑，结`.

Note that this implementation is **not synchronized**. If multiple threads access a hash map concurrently, and at least one of the threads modifies the map structurally`在结构上`, it must be synchronized externally`在外面`. (A structural modification is any operation that **adds** or **deletes** one or more mappings; merely`只不过,只是` changing the value associated with a key that an instance already contains is not a structural modification.) This is typically`通常` accomplished`完成` by synchronizing on some object that naturally encapsulates`自然地封装` the map. If no such object exists, the map should be "wrapped" using the `Collections.synchronizedMap` method. This is best done at creation time, to prevent accidental`意外地` unsynchronized access to the map:

```java
 Map m = Collections.synchronizedMap(new HashMap(...));
```

The iterators returned by all of this class's "collection view methods" are fail-fast`快速失败`: if the **map is structurally modified at any time after the iterator is created**, in any way except through the iterator's own **remove** method, the iterator will throw a `ConcurrentModificationException`. Thus, in the face of concurrent modification, the iterator fails quickly and cleanly, rather than risking arbitrary`意外的风险`, non-deterministic`不确定的` behavior at an undetermined`不确定的` time in the future.

Note that the fail-fast behavior of an iterator cannot be guaranteed as it is, generally speaking, impossible to make any hard guarantees in the presence of unsynchronized concurrent modification`在非同步的并发修改面前，没办法做到绝对的保证`. Fail-fast iterators throw `ConcurrentModificationException` on a best-effort`尽最大努力` basis. Therefore, it would be wrong to write a program that depended on this exception for its correctness: the fail-fast behavior of iterators should be used **only to detect bugs**.

This class is a member of the *Java Collections Framework*.