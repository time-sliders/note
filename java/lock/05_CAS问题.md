# ABA问题
## 什么是ABA问题
因为CAS需要在操作值得时候，检查值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A、变成了B、又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但实际上却变化了。

## ABA问题解决方法
### 1.使用版本号
ABA问题的解决思路是使用版本号，每次变量更新的时候版本号加1，那么A->B->A就会变成1A->2B->3A
### 2.jdk自带原子变量
从jdk1.5开始，jdk 的 Atomic 包里就提供了一个类 **AtomicStampedReference** 来解决 ABA 问题，这个类中的 **compareAndSet** 方法的作用就是首先检查当前引用是否等于预期引用，并且检查当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值更新为指定的新值

```java
private static class Pair<T> {
    final T reference;
    final int stamp;
		...
}

private volatile Pair<V> pair;
```

```java
public boolean compareAndSet(V   expectedReference,
                             V   newReference,
                             int expectedStamp,
                             int newStamp) {
    Pair<V> current = pair;
    return
        expectedReference == current.reference &&
        expectedStamp == current.stamp &&
        ((newReference == current.reference &&
          newStamp == current.stamp) ||
         casPair(current, Pair.of(newReference, newStamp)));
}
```

```java
private boolean casPair(Pair<V> cmp, Pair<V> val) {
    return UNSAFE.compareAndSwapObject(this, pairOffset, cmp, val);
}
```