## 介绍

volatile 是**轻量级的 synchronized**，它在多处理器开发中保证了共享变量的**可见性**。它比synchronized的**使用和执行成本会更低**，因为它不会引起线程上下文的切换和调度

## **术语定义**

1. **共享变量** 在多个线程之间能够被共享的变量被称为共享变量。共享变量包括所有的实例变量，静态变量和数组元素。他们都被存放在**堆内存**中，volatile 只作用于共享变量。
2. **内存屏障** Memory Barriers: 是一组处理器指令，用于实现对内存操作的顺序限制。
3. **缓冲行** Cache line: 缓存中可以分配的最小存储单位。处理器填写缓存线时会加载整个缓存线，需要使用多个主内存读周期。
4. **原子操作** Atomic operations: 不可中断的一个或一系列操作。
5. **缓存行填充**  cache line fill: 当处理器识别到从内存中读取操作数是可缓存的，处理器读取整个缓存行到适当的缓存（L1，L2，L3的或所有）
6. **缓存命中** cache hit: 如果进行高速缓存行填充操作的内存位置仍然是下次处理器访问的地址时，处理器从缓存中读取操作数，而不是从内存。
7. **写命中** write hit: 当处理器将操作数写回到一个内存缓存的区域时，它首先会检查这个缓存的内存地址是否在缓存行中，如果存在一个有效的缓存行，则处理器将这个操作数写回到缓存，而不是写回到内存，这个操作被称为写命中。
8. **写缺失** write misses the cache 一个有效的缓存行被写入到不存在的内存区域。

#### 实现原理

有 volatile 变量修饰的共享变量进行写操作的时候会多一行 lock 代码，lock 前缀的指令在多核处理器下会引发了两件事情。

- 将当前**处理器缓存行**的数据会写回到**系统内存**。
- 这个写回内存的操作会引起在**其他CPU里缓存了该内存地址的数据无效**。

处理器为了提高处理速度，不直接和内存进行通讯，而是先将系统内存的数据读到内部缓存（L1,L2或其他）后再进行操作，但操作完之后不知道何时会写到内存，如果对声明了 volatile 变量进行写操作，JVM就会向处理器发送一条 Lock 前缀的指令，将这个变量所在缓存行的数据写回到系统内存。但是就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题，所以在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过**嗅探在总线上传播的数据来检查自己缓存的值是不是过期**了，**当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器要对这个数据进行修改操作的时候，会强制重新从系统内存里把数据读到处理器缓存里**。

![cpu-cache-bus-memory](ref/cpu-cache-bus-memory.png)

![cpu-cache-bus-memory](ref/cpu-cache-access-time.jpeg)

举一个简单的例子

```java
i++
```

当线程运行这段代码时，首先会从主存中读取i( i = 1)，然后复制一份到CPU高速缓存中，然后CPU执行 + 1 （2）的操作，然后将数据（2）写入到告诉缓存中，最后刷新到主存中。其实这样做在单线程中是没有问题的，有问题的是在多线程中。如下：

假如有两个线程A、B都执行这个操作，按照我们正常的逻辑思维主存中的i值应该=3，但事实是这样么？分析如下：

两个线程从主存中读取i的值（1）到各自的高速缓存中，然后线程A执行+1操作并将结果写入高速缓存中，最后写入主存中，此时主存i==2,线程B做同样的操作，主存中的i仍然=2。所以最终结果为2并不是3。这种现象就是缓存一致性问题。

解决缓存一致性方案有两种：

1. 通过在总线加LOCK#锁的方式
2. 通过缓存一致性协议

但是方案1存在一个问题，它是采用一种独占的方式来实现的，即总线加LOCK#锁的话，只能有一个CPU能够运行，其他CPU都得阻塞，效率较为低下。

第二种方案，缓存一致性协议（MESI协议,修改，独占，共享，无效）它确保每个缓存中使用的共享变量的副本是一致的。其核心思想如下：当某个CPU在写数据时，如果发现操作的变量是共享变量，则会通知其他CPU告知该变量的缓存行是无效的，因此其他CPU在读取该变量时，发现其无效会重新从主存中加载数据。

## source

The Java programming language allows threads to access shared variables (synchronized). As a rule, to ensure that shared variables are consistently`一致地` and reliably`可靠地` updated, a thread should ensure that it has exclusive use of such variables by obtaining a lock that, conventionally`通常`, enforces`强制` mutual exclusion for those shared variables.**Java编程语言允许线程访问共享变量，为了确保共享变量能被准确和一致地更新，线程应该确保通过排他锁单独获得这个变量。**

**volatile是无法保证复合操作的原子性**

**volatile可以保证线程可见性且提供了一定的有序性，但是无法保证原子性。在JVM底层volatile是采用“内存屏障”来实现的。**

1. **保证可见性、不保证原子性**  <<< 重点注意
2. **禁止指令重排序**

The Java programming language provides a second mechanism, `volatile` fields, that is more convenient`方便` than locking for some purposes.

**A field may be declared `volatile`, in which case the Java Memory Model ensures that all threads see a consistent value for the variable** ([§17.4](../java_memory_model.md)).

It is a compile-time error if a `final` variable is also declared `volatile`.



**Example 8.3.1.4-1. volatile Fields**

If, in the following example, one thread repeatedly calls the method `one` (but no more than `Integer.MAX_VALUE` times in all), and another thread repeatedly calls the method `two`:

```java
class Test {
    static int i = 0, j = 0;
    static void one() { i++; j++; }
    static void two() {
        System.out.println("i=" + i + " j=" + j);
    }
}
```

then method `two` could occasionally print a value for `j` that is greater than the value of `i`, because the example includes no synchronization and, under the rules explained in [§17.4](../java_memory_model.md), the shared values of `i` and `j` might be updated out of order`非顺序更新`.

One way to prevent this out-or-order behavior would be to declare methods `one` and `two` to be `synchronized` ([§8.4.3.6](./00_synchronized.md)):

```java
class Test {
    static int i = 0, j = 0;
    static synchronized void one() { i++; j++; }
    static synchronized void two() {
        System.out.println("i=" + i + " j=" + j);
    }
}
```

This prevents method `one` and method `two` from being executed concurrently, and furthermore`此外,与此同时` guarantees that the shared values of `i` and `j` are both updated before method `one`returns. Therefore method `two` never observes a value for `j` greater than that for `i`; indeed, it always observes the same value for `i` and `j`.

Another approach would be to declare `i` and `j` to be `volatile`:

```java
class Test {
    static volatile int i = 0, j = 0;
    static void one() { i++; j++; }
    static void two() {
        System.out.println("i=" + i + " j=" + j);
    }
}
```

This allows method `one` and method `two` to be executed concurrently, but guarantees that **accesses to the shared values for `i` and `j` occur exactly as many times, and in exactly the same order, as they appear to occur during execution of the program text by each thread**. Therefore, the shared value for `j` is never greater than that for `i`, because each update to `i` must be reflected in the shared value for `i` before the update to `j` occurs. It is possible, however, that any given invocation of method `two` might observe a value for `j` that is much greater than the value observed for `i`, *because method `one` might be executed many times between the moment when method `two` fetches the value of `i` and the moment when method two fetches the value of `j`.*



----

# Java Volatile Keyword

The Java `volatile` keyword is used to mark a Java variable as "being stored in main memory". More precisely that means, that every read of a volatile variable will be read from the computer's main memory, and not from the CPU cache, and that every write to a volatile variable will be written to main memory, and not just to the CPU cache.

Actually, since Java 5 the `volatile` keyword guarantees more than just that volatile variables are written to and read from main memory. I will explain that in the following sections.



## Variable Visibility Problems

The Java `volatile` keyword guarantees visibility of changes to variables across threads. This may sound a bit abstract, so let me elaborate.

In a multithreaded application where the threads operate on non-volatile variables, each thread may copy variables from main memory into a CPU cache while working on them, for performance reasons. If your computer contains more than one CPU, each thread may run on a different CPU. That means, that each thread may copy the variables into the CPU cache of different CPUs. This is illustrated here:

![Threads may hold copies of variables from main memory in CPU caches.](http://tutorials.jenkov.com/images/java-concurrency/java-volatile-1.png)

With non-volatile variables there are no guarantees about when the Java Virtual Machine (JVM) reads data from main memory into CPU caches, or writes data from CPU caches to main memory. This can cause several problems which I will explain in the following sections.

Imagine a situation in which two or more threads have access to a shared object which contains a counter variable declared like this:

```
public class SharedObject {

    public int counter = 0;

}
```

Imagine too, that only Thread 1 increments the `counter` variable, but both Thread 1 and Thread 2 may read the `counter` variable from time to time.

If the `counter` variable is not declared `volatile` there is no guarantee about when the value of the `counter` variable is written from the CPU cache back to main memory. This means, that the `counter` variable value in the CPU cache may not be the same as in main memory. This situation is illustrated here:

![The CPU cache used by Thread 1 and main memory contains different values for the counter variable.](http://tutorials.jenkov.com/images/java-concurrency/java-volatile-2.png)

The problem with threads not seeing the latest value of a variable because it has not yet been written back to main memory by another thread, is called a "visibility" problem. The updates of one thread are not visible to other threads.



## The Java volatile Visibility Guarantee

The Java `volatile` keyword is intended to address variable visibility problems. By declaring the `counter` variable `volatile` all writes to the `counter` variable will be written back to main memory immediately. Also, all reads of the `counter` variable will be read directly from main memory.

Here is how the `volatile` declaration of the `counter` variable looks:

```
public class SharedObject {
    public volatile int counter = 0;
}
```

Declaring a variable `volatile` thus *guarantees the visibility* for other threads of writes to that variable.

In the scenario given above, where one thread (T1) modifies the counter, and another thread (T2) reads the counter (but never modifies it), declaring the `counter` variable `volatile` is enough to guarantee visibility for T2 of writes to the `counter` variable.

If, however, both T1 and T2 were incrementing the `counter` variable, then declaring the `counter` variable `volatile` would not have been enough. More on that later.



### Full volatile Visibility Guarantee

Actually, the visibility guarantee of Java `volatile` goes beyond the `volatile` variable itself. The visibility guarantee is as follows:

- If Thread A writes to a `volatile` variable and Thread B subsequently reads the same volatile variable, then all variables visible to Thread A before writing the volatile variable, will also be visible to Thread B after it has read the volatile variable.
- If Thread A reads a `volatile` variable, then all all variables visible to Thread A when reading the `volatile` variable will also be re-read from main memory.

Let me illustrate that with a code example:

```
public class MyClass {
    private int years;
    private int months
    private volatile int days;


    public void update(int years, int months, int days){
        this.years  = years;
        this.months = months;
        this.days   = days;
    }
}
```

The `udpate()` method writes three variables, of which only `days` is volatile.

The full `volatile` visibility guarantee means, that when a value is written to `days`, then all variables visible to the thread are also written to main memory. That means, that when a value is written to `days`, the values of `years` and `months` are also written to main memory.

When reading the values of `years`, `months` and `days` you could do it like this:

```
public class MyClass {
    private int years;
    private int months
    private volatile int days;

    public int totalDays() {
        int total = this.days;
        total += months * 30;
        total += years * 365;
        return total;
    }

    public void update(int years, int months, int days){
        this.years  = years;
        this.months = months;
        this.days   = days;
    }
}
```

Notice the `totalDays()` method starts by reading the value of `days` into the `total` variable. When reading the value of `days`, the values of `months` and `years` are also read into main memory. Therefore you are guaranteed to see the latest values of `days`, `months` and `years` with the above read sequence.



## Instruction Reordering Challenges

The Java VM and the CPU are allowed to reorder instructions in the program for performance reasons, as long as the semantic meaning of the instructions remain the same. For instance, look at the following instructions:

```
int a = 1;
int b = 2;

a++;
b++;
```

These instructions could be reordered to the following sequence without losing the semantic meaning of the program:

```
'
int a = 1;
a++;

int b = 2;
b++;
```

However, instruction reordering present a challenge when one of the variables is a `volatile` variable. Let us look at the `MyClass` class from the example earlier in this Java volatile tutorial:

```
public class MyClass {
    private int years;
    private int months
    private volatile int days;


    public void update(int years, int months, int days){
        this.years  = years;
        this.months = months;
        this.days   = days;
    }
}
```

Once the `update()` method writes a value to `days`, the newly written values to `years` and `months` are also written to main memory. But, what if the Java VM reordered the instructions, like this:

```
public void update(int years, int months, int days){
    this.days   = days;
    this.months = months;
    this.years  = years;
}
```

The values of `months` and `years` are still written to main memory when the `days` variable is modified, but this time it happens before the new values have been written to `months` and `years`. The new values are thus not properly made visible to other threads. The semantic meaning of the reordered instructions has changed.

Java has a solution for this problem, as we will see in the next section.

## The Java volatile Happens-Before Guarantee

To address the instruction reordering challenge, the Java `volatile` keyword gives a "happens-before" guarantee, in addition to the visibility guarantee. The happens-before guarantee guarantees that:

- **Reads from and writes to other variables cannot be reordered to occur after a write to a `volatile` variable, if the reads / writes originally occurred before the write to the `volatile` variable.**
  The reads / writes before a write to a `volatile` variable are guaranteed to "happen before" the write to the `volatile` variable. Notice that it is still possible for e.g. reads / writes of other variables located after a write to a `volatile` to be reordered to occur before that write to the `volatile`. Just not the other way around. From after to before is allowed, but from before to after is not allowed.

- **Reads from and writes to other variables cannot be reordered to occur before a read of a `volatile` variable, if the reads / writes originally occurred after the read of the `volatile` variable.** Notice that it is possible for reads of other variables that occur before the read of a `volatile` variable can be reordered to occur after the read of the `volatile`. Just not the other way around. From before to after is allowed, but from after to before is not allowed.

The above happens-before guarantee assures that the visibility guarantee of the `volatile` keyword are being enforced.



## volatile is Not Always Enough

Even if the `volatile` keyword guarantees that all reads of a `volatile` variable are read directly from main memory, and all writes to a `volatile` variable are written directly to main memory, there are still situations where it is not enough to declare a variable `volatile`.

In the situation explained earlier where only Thread 1 writes to the shared `counter` variable, declaring the `counter` variable `volatile` is enough to make sure that Thread 2 always sees the latest written value.

In fact, multiple threads could even be writing to a shared `volatile` variable, and still have the correct value stored in main memory, if the new value written to the variable does not depend on its previous value. In other words, if a thread writing a value to the shared `volatile` variable does not first need to read its value to figure out its next value.

As soon as a thread needs to first read the value of a `volatile` variable, and based on that value generate a new value for the shared `volatile` variable, a `volatile` variable is no longer enough to guarantee correct visibility. The short time gap in between the reading of the `volatile` variable and the writing of its new value, creates an [race condition](http://tutorials.jenkov.com/java-concurrency/race-conditions-and-critical-sections.html) where multiple threads might read the same value of the `volatile` variable, generate a new value for the variable, and when writing the value back to main memory - overwrite each other's values.

The situation where multiple threads are incrementing the same counter is exactly such a situation where a `volatile` variable is not enough. The following sections explain this case in more detail.

Imagine if Thread 1 reads a shared `counter` variable with the value 0 into its CPU cache, increment it to 1 and not write the changed value back into main memory. Thread 2 could then read the same `counter` variable from main memory where the value of the variable is still 0, into its own CPU cache. Thread 2 could then also increment the counter to 1, and also not write it back to main memory. This situation is illustrated in the diagram below:

![Two threads have read a shared counter variable into their local CPU caches and incremented it.](http://tutorials.jenkov.com/images/java-concurrency/java-volatile-3.png)

Thread 1 and Thread 2 are now practically out of sync. The real value of the shared `counter` variable should have been 2, but each of the threads has the value 1 for the variable in their CPU caches, and in main memory the value is still 0. It is a mess! Even if the threads eventually write their value for the shared `counter` variable back to main memory, the value will be wrong.

## When is volatile Enough?

As I have mentioned earlier, if two threads are both reading and writing to a shared variable, then using the `volatile` keyword for that is not enough. You need to use a [synchronized](http://tutorials.jenkov.com/java-concurrency/synchronized.html) in that case to guarantee that the reading and writing of the variable is atomic. Reading or writing a volatile variable does not block threads reading or writing. For this to happen you must use the `synchronized` keyword around critical sections.

As an alternative to a `synchronized` block you could also use one of the many atomic data types found in the [`java.util.concurrent` package](http://tutorials.jenkov.com/java-util-concurrent/index.html). For instance, the [`AtomicLong`](http://tutorials.jenkov.com/java-util-concurrent/atomiclong.html) or [`AtomicReference`](http://tutorials.jenkov.com/java-util-concurrent/atomicreference.html) or one of the others.

**In case only one thread reads and writes the value of a volatile variable and other threads only read the variable, then the reading threads are guaranteed to see the latest value written to the volatile variable. Without making the variable volatile, this would not be guaranteed.**

The `volatile` keyword is guaranteed to work on 32 bit and 64 variables.

## Performance Considerations of volatile

Reading and writing of volatile variables causes the variable to be read or written to main memory. Reading from and writing to main memory is more expensive than accessing the CPU cache. Accessing volatile variables also prevent instruction reordering which is a normal performance enhancement technique. Thus, you should only use volatile variables when you really need to enforce visibility of variables.