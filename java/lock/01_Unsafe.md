# Unsafe

#### 介绍

java 提供的一个可以可以绕开 JVM 直接操作内存的工具，也提供了一些底层的CAS操作的方法，一般用来实现锁，比如LockSupport, AQS 等。

## CAS

CAS 指的是现代 CPU 广泛支持的一种对内存中的共享数据进行操作的一种特殊指令。这个指令会对内存中的共享数据做**原子的读写操作**

首先，CPU 会将内存中将要被更改的数据与期望的值做比较。然后，当这两个值相等时，CPU 才会将内存中的数值替换为新的值。否则便不做操作。最后，CPU 会将旧的数值返回。

这一系列的操作是原子的。它们虽然看似复杂，但却是 Java 5 并发机制优于原有锁机制的根本。简单来说，CAS 的含义是“我认为原有的值应该是什么，如果是，则将原有的值更新为新值，否则不做修改，并告诉我原来的值是多少”。

CAS通过调用JNI（**J**ava **N**ative **I**nterface）调用实现的。JNI允许java调用其他语言，而CAS就是借助C语言来调用CPU底层指令实现的。Unsafe 是 CAS 的核心类，它提供了硬件级别的原子操作

#### Unsafe 的目的

Java 无法直接访问到操作系统底层`（如系统硬件等  原因是 JVM 屏蔽了与底层操作系统交互的细节，所以java程序无法直接访问操作系统)`，为此 Java 使用 native 方法来扩展  Java 程序的功能。**Unsafe 类提供了硬件级别的原子操作，提供了一些绕开JVM的，直接操作底层的功能，由此提高效率**。

#### 为什么不建议直接在代码中使用 Unsafe

* Unsafe 的不少方法中必须提供原始地址(内存地址)和被替换对象的地址，偏移量要自己计算，一旦出现问题就是JVM崩溃级别的异常，会导致**整个JVM实例崩溃**，表现为应用程序直接 crash 掉。
* Unsafe 提供的**直接内存**访问的方法中使用的内存**不受 JVM 管理**(无法被 GC )，需要手动管理，一旦出现疏忽很有可能成为**内存泄漏**的源头。

#### 如何在业务代码中获取 Unsafe 对象

Unsafe 类被设计成单例，且构造方法私有化，getUnsafe 方法又限制了调用方的类加载器，导致应用程序无法直接创建 Unsafe 实例。代码如下：

```java
static {
    Reflection.registerMethodsToFilter(Unsafe.class, "getUnsafe");
}
private Unsafe() {}  // 构造函数私有化

private static final Unsafe theUnsafe = new Unsafe();
private static final jdk.internal.misc.Unsafe theInternalUnsafe = jdk.internal.misc.Unsafe.getUnsafe();

@CallerSensitive
public static Unsafe getUnsafe() {
    Class<?> caller = Reflection.getCallerClass();
		// 只允许系统类加载器加载的类调用该方法获取实例
    if (!VM.isSystemDomainLoader(caller.getClassLoader()))  
        throw new SecurityException("Unsafe");
    return theUnsafe;
}
```

在应用程序中，可以通过反射的方式获取 Unsafe 类的 theUnsafe 属性，即可拿到 Unsafe 实例。代码如下：

```java
Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
theUnsafe.setAccessible(true);
Unsafe unsafe = (Unsafe) theUnsafe.get(null);
```

#### Unsafe 的核心功能

* **线程的挂起和恢复**

  * `public native void unpark(Object thread);`
    * 释放被 `park` 创建的在一个线程上的阻塞。**如果线程并没有被阻塞，那么下一个`park`将不会被阻塞**。这个操作是不安全的，因此必须保证线程是存活的。从Java代码中判断一个线程是否存活的是显而易见的，但是从 native 代码中这几乎是不可能自动完成的。
  * `public native void park(boolean isAbsolute, long time);`
    * 阻塞当前线程直到一个 `unpark` 方法出现(被调用)、或者 `unpark` 方法已经调用过、线程被中断或者 time 时间到期(也就是阻塞超时)。在 time 非零的情况下，如果 `isAbsolute` 为 true，time 是相对于新纪元之后的毫秒数，否则 time 表示纳秒。**这个方法执行时也可能不合理地返回(没有具体原因)**。并发包java.util.concurrent中的框架对线程的挂起操作被封装在LockSupport 类中，LockSupport 类中有各种版本 pack 方法，但最终都调用了 `Unsafe#park()` 方法。
    * 常用方式 `park(false, 0L)`

* **多线程同步**

  **主要包括监视器锁定、解锁以及 CAS 相关的方法**。

  * `public native void monitorEnter(Object o);`
    * 锁定对象，必须通过`monitorExit`方法才能解锁。此方法是可以重入的，也就是可以多次调用，然后通过多次调用`monitorExit`进行解锁。
  * `public native void monitorExit(Object o);`
    * 解锁对象，前提是对象必须已经调用`monitorEnter`进行加锁，否则抛出*IllegalMonitorStateException*异常。
  * `public native boolean tryMonitorEnter(Object o);`
    * 尝试锁定对象，如果加锁成功返回true，否则返回false。必须通过`monitorExit`方法才能解锁。
  * `public final native boolean compareAndSwapObject(Object o, long offset, Object expected, Object x);`
    * 针对 Object 对象进行 CAS 操作。即是对应 Java 变量引用 o，原子性地更新 o 中偏移地址为 offset 的属性的值为 x，当且仅的偏移地址为 offset 的属性的当前值为 expected 才会更新成功返回true，否则返回false。
    * 类似的方法有`compareAndSwapInt`和`compareAndSwapLong`，在Jdk8中基于 **CAS** 扩展出来的方法有`getAndAddInt`、`getAndAddLong`、`getAndSetInt`、`getAndSetLong`、`getAndSetObject`，它们的作用都是：**通过CAS设置新的值，返回旧的值**。

* **内存管理**

  使用 Unsafe 的内存管理技术，可以实现一些超大数组等情况，从而避免 JVM 堆内存限制。

  * `public native int addressSize();`
    * 获取本地内存的页数，此值为2的幂次方。
  * `public native long allocateMemory(long bytes);`
    * 分配一块新的本地内存，通过bytes指定内存块的大小(单位是byte)，返回新开辟的内存的地址。如果内存块的内容不被初始化，那么它们一般会变成内存垃圾。生成的本机指针永远不会为零，并将对所有值类型进行对齐。可以通过`freeMemory`方法释放内存块，或者通过`reallocateMemory`方法调整内存块大小。bytes值为负数或者过大会抛出 *IllegalArgumentException* 异常，如果系统拒绝分配内存会抛出 *OutOfMemoryError* 异常。

* **类、对象和变量相关方法**

  主要包括类的非常规实例化、基于偏移地址获取或者设置变量的值、基于偏移地址获取或者设置数组元素的值等。

  * `public native Object getObject(Object o, long offset);` 
    * 通过给定的Java变量获取引用值。这里实际上是获取一个Java对象o中，获取偏移地址为offset的属性的值，此方法可以突破修饰符的抑制，也就是无视private、protected和default修饰符。类似的方法有`getInt`、`getDouble`等等。
  * `public native long staticFieldOffset(Field f);`
    * 返回给定的静态属性在它的类的存储分配中的位置(偏移地址)。不要在这个偏移量上执行任何类型的算术运算，它只是一个被传递给不安全的堆内存访问器的cookie。注意：这个方法仅仅针对静态属性。 与之对应的另外一个方法是 `objectFieldOffset` 该方法只能用于非静态属性。
  * `public native Object allocateInstance(Class<?> cls) throws InstantiationException;`
    * 通过Class对象创建一个类的实例，不需要调用其构造函数、初始化代码、JVM安全检查等等。同时，它抑制修饰符检测，也就是即使构造器是private修饰的也能通过此方法实例化。
    * 可以通过该方法，创建对象实例，且不会调用 Class 上定义的构造器中的一些初始化方法，得到一个完全的"空"对象，避免初始化。

  ##### 示例：DirectByteBuffer 分配通过 Unsafe 管理内存

  ```java
  DirectByteBuffer(int cap) {                   // package-private
  
      super(-1, 0, cap, cap);
      boolean pa = VM.isDirectMemoryPageAligned();
      int ps = Bits.pageSize();
      long size = Math.max(1L, (long)cap + (pa ? ps : 0));
      Bits.reserveMemory(size, cap);
  
      long base = 0;
      try {
          base = unsafe.allocateMemory(size); //使用Unsafe分配内存
      } catch (OutOfMemoryError x) {
          Bits.unreserveMemory(size, cap);
          throw x;
      }
      unsafe.setMemory(base, size, (byte) 0); //使用Unsafe设置内存固定值
      if (pa && (base % ps != 0)) {
          // Round up to page boundary
          address = base + ps - (base & (ps - 1));
      } else {
          address = base;
      }
      cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
      att = null;
  }
  ```
