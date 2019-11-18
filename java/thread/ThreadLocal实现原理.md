## ThreadLocal.get()

```java
/**
 * Returns the value in the current thread's copy of this
 * thread-local variable.  If the variable has no value for the
 * current thread, it is first initialized to the value returned
 * by an invocation of the {@link #initialValue} method.
 *
 * @return the current thread's value of this thread-local
 */
public T get() {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 获取当前线程对象的 threadLocals 属性
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
  return t.threadLocals;
}
```

从上面的方法可以看出，get 方法是从 `Thread.currentThread().threadlocals.getEntry(threadlocal)` 来获取的

我们继续进入 Thread 类可以看到线程的 threadlocals 属性实际上是一个 ThreadLocal.ThreadLocalMap 对象。

```java
static class ThreadLocalMap {

    /**
     * Entry 是该 hashmap 的元素，它继承了 WeakReference 类，并且使用 key
     * 来做为 WeakReference 的 referent，这里 key 都是 threadlocal 对象本身
     * 这意味着，如果 threadlocal 对象不再使用，那么将有可能会被 gc 回收
     */
    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;
        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    private Entry[] table;// hash 表，存放数据，大小必须为 2^n，自动扩容
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1); // 计算要放到哪一个槽
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }
  
 
```

上面方法可以简单看出该 map 本质上是一个自动扩容的数组，由于实现 WeakReference ，所以一旦 Threadlocals 对象不可达，则放进去的 key 会被自动回收，后续，该 Entry 也会被回收，我们继续看他的 getEntry 方法

```java
/**
 * Get the entry associated with key.  This method
 * itself handles only the fast path: a direct hit of existing
 * key. It otherwise relays to getEntryAfterMiss.  This is
 * designed to maximize performance for direct hits, in part
 * by making this method readily inlinable.
 *
 * @param  key the thread local object
 * @return the entry associated with key, or null if no such
 */
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);// 计算当前threadlocal key 是存放在哪一个 index 上
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;// 找到了
    else
        return getEntryAfterMiss(key, i, e); // 没找到
}
```

继续往下看没找到要怎么处理

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);//清除掉那些 key 已经被 WeakReference 机制回收掉的对象
        else
            i = nextIndex(i, len);// 获取数组的下一个元素，如果越界，则从0开始处理
        e = tab[i];
    }
    return null;
}
```

这里原理是从数组第 i 个位置开始往后找(找到最后一个就继续重0重新开始)，直到找到 k == key 的元素返回，如果遇到 空节点，就停止寻找，返回 null。这里有一个疑问，为什么当前节点找不到要继续往后找？或者说，什么时候会把当前节点的元素移到后续节点里面去？这里猜测是在Set 或者 扩容的时候做的，那么我们先看下 set 算法

```java
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1); // 计算要放入的位置

    /*
     * 同 hashMap 处理的方式不一样，如果有多个元素根据它的 hash 算法，定位到同一个 slot
     * 那么 后续放入的元素，就在当前数组上的 nextIndex 位置，即往后延
     */
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) { // 如果已经存在，则覆盖原数据
            e.value = value;
            return;
        }
      
				// 如果元素已经被 gc 回收，则需要重新找到原来存放数据的位置，然后替换数据
        if (k == null) {
            // 这个方法是在可能的 run 区间找到元素并替换元素，同时还有一定的清理功能
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

由于 ThreadLocal 对象是天然线程安全的（Thread只能是一个单线程），所以根本**不存在并发问题**。