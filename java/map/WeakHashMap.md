在了解 WeakHashMap 之前，请先了解一下 [WeakReference](../jvm/gc/Java-WeakReference.etc.md)

# source code

WeakHashMap的关键点是 Entry 的实现

```java
/**
 * The entries in this hash table extend WeakReference, using its main ref
 * field as the key.
 */
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
    V value; // 注意这里没有 key 属性
    final int hash;
    Entry<K,V> next;

    /**
     * Creates new entry.
     */
    Entry(Object key, V value,
          ReferenceQueue<Object> queue,
          int hash, Entry<K,V> next) {
        super(key, queue);
        this.value = value;
        this.hash  = hash;
        this.next  = next;
    }

    @SuppressWarnings("unchecked")
    public K getKey() {
        return (K) WeakHashMap.unmaskNull(get());
    }

		// ${其他代码}
}
```

从上面的代码里面可以看到，Entry对象继承了WeakReference，WeakReference 中的 referent 字段存的就是 key，所以在 Entry 对象中并没有 Key 属性（否则的话就会有一个 key 的强引用导致 GC 无法自动回收）。

这样我们就可以知道，在gc的时候，key 会被自动 gc 回收，然后根据 WeakReference 的生命机制，WeakReference 对象本身，也就是这里的 Entry 对象，在GC之后，会被一个**单线程**添加到 referenceQueue 中。（注意这里不是马上添加到，所以key被GC之后，实际上需要过小一段时间才会从map上移走）

我们在继续看下 expungeStaleEntries 方法

```java
/**
 * Expunges stale entries from the table.
 */
private void expungeStaleEntries() {
    for (Object x; (x = queue.poll()) != null; ) {
        synchronized (queue) {
            @SuppressWarnings("unchecked")
                Entry<K,V> e = (Entry<K,V>) x;
            int i = indexFor(e.hash, table.length);

            Entry<K,V> prev = table[i];
            Entry<K,V> p = prev;
            while (p != null) {
                Entry<K,V> next = p.next;
                if (p == e) {
                    if (prev == e)
                        table[i] = next;
                    else
                        prev.next = next;
                    // Must not null out e.next;
                    // stale entries may be in use by a HashIterator
                    e.value = null; // Help GC
                    size--;
                    break;
                }
                prev = p;
                p = next;
            }
        }
    }
}
```

可以看到 该方法里面从 referenceQueue 中获取已经被gc 回收的对象，然后进行清理（即将该 Entry 从map 中移除）。而该方法在 WeakHashMap 的所有方法中均有被调用，从而完成已经被 gc 掉的 key 所对应的 Entry 自动清除的动作。