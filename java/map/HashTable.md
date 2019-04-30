# sourcecode

## 主要属性

```java
private transient Entry<?,?>[] table;// The hash table data. hash 表
private transient int count;//The total number of entries in the hash table.
// (int)(capacity * loadFactor).)
private int threshold;// The table is rehashed when its size exceeds this threshold.
private float loadFactor;//The load factor for the hashtable.
/**
 * The number of times this Hashtable has been structurally modified
 * Structural modifications are those that change the number of entries in
 * the Hashtable or otherwise modify its internal structure (e.g.,
 * rehash).  This field is used to make iterators on Collection-views of
 * the Hashtable fail-fast.  (See ConcurrentModificationException).
 */
private transient int modCount = 0;
```

## 核心方法

### get

```java
/**
 * Returns the value to which the specified key is mapped,
 * or {@code null} if this map contains no mapping for the key.
 *
 * <p>More formally, if this map contains a mapping from a key
 * {@code k} to a value {@code v} such that {@code (key.equals(k))},
 * then this method returns {@code v}; otherwise it returns
 * {@code null}.  (There can be at most one such mapping.)
 *
 * @param key the key whose associated value is to be returned
 * @return the value to which the specified key is mapped, or
 *         {@code null} if this map contains no mapping for the key
 * @throws NullPointerException if the specified key is null
 * @see     #put(Object, Object)
 */
@SuppressWarnings("unchecked")
public synchronized V get(Object key) {
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
  	/*可以看到hash算法只是用 hashcode 的低16位 简单的求余表长*/
    int index = (hash & 0x7FFFFFFF) % tab.length;
    for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            return (V)e.value;
        }
    }
    return null;
}
```

### put

```java
/**
 * Maps the specified {@code key} to the specified
 * {@code value} in this hashtable. Neither the key nor the
 * value can be {@code null}. <p>
 *
 * The value can be retrieved by calling the {@code get} method
 * with a key that is equal to the original key.
 *
 * @param      key     the hashtable key
 * @param      value   the value
 * @return     the previous value of the specified key in this hashtable,
 *             or {@code null} if it did not have one
 * @exception  NullPointerException  if the key or value is
 *               {@code null}
 * @see     Object#equals(Object)
 * @see     #get(Object)
 */
public synchronized V put(K key, V value) {
    // Make sure the value is not null
    if (value == null) {
        throw new NullPointerException();
    }

    // Makes sure the key is not already in the hashtable.
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> entry = (Entry<K,V>)tab[index];
  	// 如果 hash 表上已经有这个元素了，那么直接修改这个元素，并返回旧值
    for(; entry != null ; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            entry.value = value;
            return old;
        }
    }
		// 如果hashtable 本来没有这个元素，则新增一个 Entry
    addEntry(hash, key, value, index);
    return null;
}
```

#### addEntry

```java
private void addEntry(int hash, K key, V value, int index) {
    Entry<?,?> tab[] = table;
    if (count >= threshold) {
        // Rehash the table if the threshold is exceeded
        rehash();

      	// 重新获取rehash后新表的数据
        tab = table;
        hash = key.hashCode();
        index = (hash & 0x7FFFFFFF) % tab.length;
    }

    // Creates the new entry.
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>) tab[index];
    tab[index] = new Entry<>(hash, key, value, e);
    count++;
    modCount++;
}
```

#### rehash

```java
/**
 * Increases the capacity of and internally reorganizes this
 * hashtable, in order to accommodate and access its entries more
 * efficiently.  This method is called automatically when the
 * number of keys in the hashtable exceeds this hashtable's capacity
 * and load factor.
 */
@SuppressWarnings("unchecked")
protected void rehash() {
    int oldCapacity = table.length;
    Entry<?,?>[] oldMap = table;

    // overflow-conscious code
    int newCapacity = (oldCapacity << 1) + 1;//同 hashmap 不同，这里不需要是2的整数次幂
    if (newCapacity - MAX_ARRAY_SIZE > 0) { // 超过最大容量
        if (oldCapacity == MAX_ARRAY_SIZE)
            // Keep running with MAX_ARRAY_SIZE buckets
            return;
        newCapacity = MAX_ARRAY_SIZE;
    }
    Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

    modCount++;
    threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    table = newMap;

  	// 依次将 hashtable 旧表的数据移到新表上
    for (int i = oldCapacity ; i-- > 0 ;) {
        for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
            Entry<K,V> e = old;
            old = old.next;

            int index = (e.hash & 0x7FFFFFFF) % newCapacity;
            e.next = (Entry<K,V>)newMap[index];
            newMap[index] = e;
        }
    }
}
```

### remove

```java
/**
 * Removes the key (and its corresponding value) from this
 * hashtable. This method does nothing if the key is not in the hashtable.
 *
 * @param   key   the key that needs to be removed
 * @return  the value to which the key had been mapped in this hashtable,
 *          or {@code null} if the key did not have a mapping
 * @throws  NullPointerException  if the key is {@code null}
 */
public synchronized V remove(Object key) {
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>)tab[index];
    for(Entry<K,V> prev = null ; e != null ; prev = e, e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            if (prev != null) {
                prev.next = e.next;
            } else {
                tab[index] = e.next;
            }
            modCount++;
            count--;
            V oldValue = e.value;
            e.value = null;
            return oldValue;
        }
    }
    return null;
}
```

**通过观察可以看到，整个HashTable 通过 synchronized 关键字进行并发控制，每一个操作都会锁整个 HashTable 对象。另外 hash & resize 方法设计的相对比较简单，在面对大表时的 rehash 方法耗时比较长。**

# javadoc

This class implements a hash table, which maps keys to values. Any **non-null object** can be used as a key or as a value.

To successfully store and retrieve objects from a hashtable, the objects used as **keys must implement the hashCode method and the equals method.**

An instance of `Hashtable` has two parameters that affect its performance: **initial capacity** and **load factor**. The **capacity is the number of buckets** in the hash table, and the initial capacity is simply the capacity at the time the hash table is created. Note that the hash table is open: in the case of a "hash collision", a single bucket stores multiple entries, which **must be searched sequentially**. The load factor is a measure of how full the hash table is allowed to get before its capacity is automatically increased. The *initial capacity* and *load factor* parameters are merely`只不过` hints to提示 the implementation. The exact details as to when and whether the `rehash` method is invoked are implementation-dependent.

Generally, the default load factor (.75) offers a good tradeoff between time and space costs. Higher values decrease the space overhead but increase the time cost to look up an entry (which is reflected in most Hashtable operations, including `get` and `put`).

The `initial capacity` controls a tradeoff between wasted space and the need for rehash operations, which are time-consuming. No `rehash` operations will ever occur if the *initial capacity* is greater than the maximum number of entries the `Hashtable` will contain divided by its *load factor*. However, setting the initial capacity too high can waste space.

If many entries are to be made into a `Hashtable`, creating it with a sufficiently`充分的` large capacity may allow the entries to be inserted more efficiently than letting it perform automatic rehashing as needed to grow the table.
This example creates a hashtable of numbers. It uses the names of the numbers as keys:   

~~~java
Hashtable<String, Integer> numbers = new Hashtable<String, Integer>();
numbers.put("one", 1);
numbers.put("two", 2);
numbers.put("three", 3);
~~~
To retrieve a number, use the following code:

~~~java
Integer n = numbers.get("two");
if (n != null) {
		System.out.println("two = " + n);
}
~~~

The iterators returned by the `iterator` method of the collections returned by all of this class's "collection view methods" are **fail-fast**: **if the Hashtable is structurally modified at any time after the iterator is created, in any way except through the iterator's own remove method, the iterator will throw a `ConcurrentModificationException`.** Thus, in the face of concurrent modification, the iterator fails quickly and cleanly, rather than risking arbitrary, non-deterministic behavior at an undetermined time in the future. The Enumerations returned by Hashtable's keys and elements methods are not fail-fast; if the Hashtable is structurally modified at any time after the enumeration is created then the results of enumerating are undefined.

**Note that the fail-fast behavior of an iterator cannot be guaranteed as it is**, generally speaking, impossible to make any hard guarantees in the presence of unsynchronized concurrent modification. Fail-fast iterators throw ConcurrentModificationException on a best-effort basis. Therefore, it would be wrong to write a program that depended on this exception for its correctness: the fail-fast behavior of iterators should be used only to detect bugs.

As of the Java 2 platform v1.2, this class was retrofitted`改型` to implement the `Map` interface, making it a member of the Java Collections Framework. Unlike the new collection implementations, Hashtable is `synchronized`. If a thread-safe implementation is not needed, it is recommended to use `HashMap` in place of Hashtable. If a thread-safe highly-concurrent implementation is desired, then it is recommended to use `java.util.concurrent.ConcurrentHashMap` in place of Hashtable.