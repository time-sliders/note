# synchronized

## 17.1. Synchronization 

[Synchronization](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.1) 

The Java programming language provides multiple mechanisms for communicating between threads. The most basic of these methods is ***synchronization***, which is implemented using ***monitors***. **Each object in Java is associated with a monitor, which a thread can *lock* or *unlock***. Only one thread at a time may hold a lock on a monitor. Any other threads attempting to lock that monitor are blocked until they can obtain a lock on that monitor. A thread *t* may lock a particular monitor multiple times; each **unlock** reverses the effect of one **lock** operation.

The `synchronized` statement computes a **reference to an object**; it then attempts to perform a lock action **on that object's monitor** and does not proceed further`进一步执行` until the lock action has successfully completed. After the lock action has been performed, the body of the `synchronized` statement is executed. If execution of the body is ever completed, either normally`正常地` or abruptly`突然地`, an `unlock` action is automatically performed on that same monitor.

A `synchronized` method automatically **performs a lock action when it is invoked**; its body is not executed until the lock action has successfully completed. **If the method is an instance method, it locks the monitor associated with the instance for which it was invoked** (that is, the object that will be known as `this` during execution of the body of the method). **If the method is `static`, it locks the monitor associated with the `Class` object that represents the class in which the method is defined**. If execution of the method's body is ever completed, either normally or abruptly, an unlock action is automatically performed on that same monitor.

The Java programming language neither prevents nor requires detection`检测` of deadlock conditions. Programs where threads hold (directly or indirectly) locks on multiple objects should use conventional techniques for deadlock avoidance`避免死锁的常规`, creating higher-level locking primitives`原语` that do not deadlock, if necessary.

Other mechanisms, such as reads and writes of `volatile` variables and the use of classes in the `java.util.concurrent` package, provide alternative`可供替代的` ways of synchronization.

## 17.2. Wait Sets and Notification

Every object, in addition to having an associated monitor, has an associated *wait set*. **A wait set is a set of threads**.

When an object is first created, its wait set is empty. Elementary actions`基础操作` that add threads to and remove threads from wait sets are atomic`原子性的`. Wait sets are manipulated`控制` solely`单独` through the methods`Object.wait`, `Object.notify`, and `Object.notifyAll`.

**Wait set manipulations操作 can also be affected by the interruption status of a thread**, and by the `Thread` class's methods dealing with interruption. Additionally`另外`, the `Thread` class's methods for sleeping and joining other threads have properties derived`来源` from those of wait and notification actions.

## 14.19. The `synchronized` Statement

[The `synchronized` Statement](https://docs.oracle.com/javase/specs/jls/se12/html/jls-14.html#jls-14.19)

A `synchronized` statement acquires a mutual-exclusion lock ([§17.1](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.1)) on behalf of`代表` the executing thread, executes a block, then releases the lock. While the executing thread owns the lock, no other thread may acquire the lock.

SynchronizedStatement:

```java
synchronized ( Expression ) Block
```

**The type of *Expression* must be a reference type, or a compile-time error occurs.**

A `synchronized` statement is executed by first evaluating`求…值` the *Expression*. Then:

- If evaluation of the *Expression* completes abruptly`突然地` for some reason, then the `synchronized` statement completes abruptly for the same reason.
- **Otherwise, if the value of the *Expression* is `null`, a `NullPointerException` is thrown.**
- Otherwise, let the non-`null` value of the *Expression* be `V`. The executing thread locks the monitor associated with `V`. Then the *Block* is executed, and then there is a choice:
  - If execution of the *Block* completes normally, then the monitor is unlocked and the `synchronized` statement completes normally.
  - If execution of the *Block* completes abruptly for any reason, then the monitor is unlocked and the `synchronized` statement completes abruptly for the same reason.

The locks acquired by `synchronized` statements are the same as the locks that are acquired implicitly`隐含地` by `synchronized` methods. A single thread may acquire a lock more than once.

**Acquiring the lock associated with an object does not in itself prevent other threads from accessing fields of the object or invoking un-`synchronized` methods on the object**`获取与对象关联的锁本身并不阻止其他线程访问对象的字段或对对象调用未同步的方法`. Other threads can also use `synchronized` methods or the `synchronized` statement in a conventional manner`常规的方式` to achieve`实现` mutual exclusion.

**Example 14.19-1. The synchronized Statement**

```java
class Test {
    public static void main(String[] args) {
        Test t = new Test();
        synchronized(t) {
            synchronized(t) {
                System.out.println("made it!");
            }
        }
    }
}
```

This program produces the output:

```
made it!
```

Note that this program would deadlock if a single thread were not permitted to lock a monitor more than once.

## 8.4.3.6. `synchronized` Methods

[`synchronized` Methods](<https://docs.oracle.com/javase/specs/jls/se12/html/jls-8.html#jls-8.4.3.6>)

A `synchronized` method acquires a monitor  before it executes.

* **For a class (`static`) method, the monitor associated with the `Class` object for the method's class is used.**

* **For an instance method, the monitor associated with `this` (the object for which the method was invoked) is used.**

**这里可以看出，同一个类的所有 synchronized static 方法会互相阻塞，同一个实例的 synchronized 方法会互相阻塞。**

**Example 8.4.3.6-1. synchronized Monitors**

These are the same monitors that can be used by the `synchronized` statement ([§14.19](https://docs.oracle.com/javase/specs/jls/se12/html/jls-14.html#jls-14.19)).

Thus, the code:

```java
class Test {
    int count;
    synchronized void bump() {
        count++;
    }
    static int classCount;
    static synchronized void classBump() {
        classCount++;
    }
}
```

has exactly the same effect as:

```java
class BumpTest {
    int count;
    void bump() {
        synchronized (this) { count++; }
    }
    static int classCount;
    static void classBump() {
        try {
            synchronized (Class.forName("BumpTest")) {
                classCount++;
            }
        } catch (ClassNotFoundException e) {}
    }
}
```



**Example 8.4.3.6-2. synchronized Methods**

```java
public class Box {
    private Object boxContents;
    public synchronized Object get() {
        Object contents = boxContents;
        boxContents = null;
        return contents;
    }
    public synchronized boolean put(Object contents) {
        if (boxContents != null) return false;
        boxContents = contents;
        return true;
    }
}
```

This program defines a class which is designed for concurrent use. Each instance of the class `Box` has an instance variable `boxContents` that can hold a reference to any object. You can put an object in a `Box` by invoking `put`, which returns `false` if the box is already full. You can get something out of a `Box` by invoking `get`, which returns a null reference if the box is empty.

If `put` and `get` were not `synchronized`, and two threads were executing methods for the same instance of `Box` at the same time, then the code could misbehave. It might, for example, lose track of an object because two invocations to `put` occurred at the same time.





# JVM 对 Synchronized 的优化

java中锁的优化 -- JVM对synchronized的优化  

## 锁消除 
概念：JVM在JIT编译(即时编译)时，通过对运行上下文的扫描，去除掉那些不可能发生共享资源竞争的锁，从而节省了线程请求这些锁的时间。 

>举例： 
>StringBuffer 的 append 方法是一个同步方法，如果 StringBuffer 类型的变量是一个局部变量，则该变量就不会被其它线程所使用，即对局部变量的操作是不会发生线程不安全的问题。 
>在这种情景下，JVM会在JIT编译时自动将append方法上的锁去掉。 相当于StringBuilder

## 锁粗化 
概念：将多个连续的加锁、解锁操作连接在一起，扩展成一个范围更大的锁，即将加锁的粒度放大。
>举例：在for循环里的加锁/解锁操作，一般需要放到for循环外。 

## 使用偏向锁和轻量级锁
说明： 
1. java6 为了减少获取锁和释放锁带来的性能消耗，引入了偏向锁和轻量级锁 
2. 锁一共有4种状态，级别从低到高依次是：无锁状态、偏向锁、轻量级锁、重量级锁
3. 锁的状态会随着竞争情况逐渐升级，并且只可以升级而不能降级
          
### 偏向锁
**背景**：大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。 
**概念**：核心思想就是锁会偏向第一个获取它的线程，如果在接下来的执行过程中没有其它的线程获取该锁，则持有偏向锁的线程永远不需要同步。 
**目的**：偏向锁实际上是一种优化锁，其目的是为了**减少数据在无竞争情况下的性能损耗**。 
**原理**：

1. 当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID。
2. 以后该线程在进入和退出同步块时就不需要进行CAS操作来加锁和解锁，只需简单地判断一下对象头的 Mark Word 里是否存储着指向当前线程的偏向锁。

**偏向锁的获取**： 
1. 访问 Mark Word 中偏向锁的标识位是否为1，如果是1，则确定为偏向锁。 
>说明： 
>
>如果偏向锁的标识位为0，说明此时是处于无锁状态，则当前线程通过 CAS 操作尝试获取偏向锁，如果获取锁成功，则将 Mark Word 中的偏向 线程ID 设置为当前 线程ID；并且将偏向标识位设为 1。 
>
>如果偏向锁的标识位不为1，也不为0(此时偏向锁的标识位没有值)，说明发生了竞争，偏向锁已经膨胀为轻量级锁，这时使用 CAS 操作尝试获得锁。
>
>关于 MarkWord
>
>对象实例由对象头、实例数据组成，其中对象头包括markword和类型指针，如果是数组，还包括数组长度;
>
>| 类型     | 32位JVM | 64位JVM |
>| -------- | ------- | ------: |
>| markword | 32bit   |   64bit |
>| 类型指针 | 32bit |64bit ，开启指针压缩时为32bit |
>| 数组长度 | 32bit |32bit |
>
>
>
>| 偏向锁标识位 | 锁标识位 | 锁状态   | 存储内容                     |
>| :----------- | :------- | :------- | :--------------------------- |
>| 0            | 01       | 未锁定   | hash code(31),年龄(4)        |
>| 1            | 01       | 偏向锁   | 线程ID(54),时间戳(2),年龄(4) |
>| 无           | 00       | 轻量级锁 | 栈中锁记录的指针(64)         |
>| 无           | 10       | 重量级锁 | monitor的指针(64)            |
>| 无           | 11       | GC标记   | 空，不需要记录信息           |

2. 如果是偏向锁，则判断 Mark Word 中的偏向线程ID是否指向当前线程，如果偏向线程ID指向当前线程，则表明当前线程已经获取到了锁；
3. 如果偏向 线程ID 并未指向当前线程，则通过 CAS 操作尝试获取偏向锁，如果获取锁成功，则将 Mark Word 中的偏向 线程ID 设置为当前 线程ID； 
4. 如果 CAS 获取偏向锁失败，则表示有竞争。当到达全局安全点时(在这个时间点上没有正在执行的字节码)，获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码。

**偏向锁的释放**

1. 当其它的线程尝试获取偏向锁时，持有偏向锁的线程才会释放偏向锁。
2. 释放偏向锁需要等待全局安全点(在这个时间点上没有正在执行的字节码)。
>过程： 
	首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着，如果线程不处于活动状态，则将对象头设置成无锁状态，如果线程还活着，说明此时发生了竞争，则偏向锁升级为轻量级锁，然后刚刚被暂停的线程会继续往下执行同步代码。  

**优点**：加锁和解锁不需要额外的消耗，和执行非同步方法相比仅存在纳秒级的差距 
**缺点**：如果线程间存在锁竞争，锁撤销会带来额外的消耗。 

> 说明： 
>
> 1. 偏向锁默认在应用程序启动几秒钟之后才激活
> 2. 可以通过设置 -XX:BiasedLockingStartupDelay=0 来关闭延迟
> 3. 可以通过设置 -XX:-UseBiasedLocking=false 来关闭偏向锁，程序默认会进入轻量级锁状态。(如果应用程序里的锁大多情况下处于竞争状态，则应该将偏向锁关闭)

## 轻量级锁
**原理**

1. 当使用轻量级锁(锁标识位为00)时，线程在执行同步块之前，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的Mark Word复制到锁记录中(注:锁记录中的标识字段称为Displaced Mark Word)。 
2. **将对象头中的 MarkWord 复制到栈桢中的锁记录中之后**，**虚拟机将尝试使用CAS将对象头中Mark Word替换为指向该线程虚拟机栈中锁记录的指针**，此时如果没有线程占有锁或者没有线程竞争锁，则当前线程成功获取到锁，然后执行同步块中的代码。 
3. 如果在获取到锁的线程执行同步代码的过程中，另一个线程也完成了栈桢中锁记录的创建，并且已经将对象头中的 MarkWord 复制到了自己的锁记录中，然后尝试使用 CAS 将对象头中的 MarkWord 修改为指向自己的锁记录的指针，但是由于之前获取到锁的线程已经将对象头中的MarkWord修改过了(并且现在还在执行同步体中的代码,即仍然持有着锁)，所以此时对象头中的MarkWord与当前线程锁记录中MarkWord的值不同，导致CAS操作失败，然后该线程就会**不停地循环使用CAS操作试图**将对象头中的MarkWord替换为自己锁记录中MarkWord的值，(**当循环次数或循环时间达到上限时停止循环**)如果在循环结束之前CAS操作成功，那么该线程就可以成功获取到锁，**如果循环结束之后依然获取不到锁，则锁获取失败，对象头中的MarkWord会被修改为指向重量级锁的指针，然后这个获取锁失败的线程就会被挂起，阻塞了**。 
4. **当持有锁的那个线程执行完同步体之后，使用 CAS 操作将对象头中的 MarkWord 还原为最初的状态时(将对象头中指向锁记录的指针替换为Displaced Mark Word )，发现 MarkWord 已被修改为指向重量级锁的指针，因此 CAS 操作失败，该线程会释放锁并唤起阻塞等待的线程，开始新一轮夺锁之争，而此时，轻量级锁已经膨胀为重量级锁，所有竞争失败的线程都会阻塞，而不是自旋**。  

 >自旋锁
 >1. 所谓自旋锁，就是让没有获得锁的进程自己运行一段时间自循环(默认开启)，但是不挂起线程
 >2. 自旋的代价就是该线程会一直占用处理器如果锁占用的时间很短，自旋等待的效果很好，反之，自旋锁会消耗大量处理器资源。
 >3. 因此，自旋的等待时间必须有一定限度，超过限度还没有获得锁，就要挂起线程。
 >
 >优点：在没有多线程竞争的前提下，减少传统的重量级锁带来的性能损耗
 >缺点：竞争的线程如果始终得不到锁，自旋会消耗cpu
 >应用：追求响应时间，同步块执行速度非常快。

## 重量级锁
说明

1. java6 之前的 synchronized 属于重量级锁，效率低下，因为 monitor 是依赖操作系统的 Mutex Lock(互斥量) 来实现的
2. 操作系统实现线程之间的切换需要从用户态转换到核心态，这个状态之间的转换需要相对较长的时间，时间成本相对较高
3. 在互斥状态下，没有得到锁的线程会被挂起阻塞，而挂起线程和恢复线程的操作都需要从用户态转入内核态中完成 

优点：线程竞争不使用自旋，不会消耗 cpu
缺点：线程阻塞，响应时间缓慢
应用：追求吞吐量，同步块执行速度较长 



总结
java synchronized 绝大部分情况下都是单线程加锁，出现冲突的情况比较少，所以 synchronized 一开始是偏向锁，会在加锁对象的头部 MarkWord 中记录持有锁的线程的ID，以后该线程在进入和退出同步块时就不需要进行 CAS 操作来加锁和解锁（解锁的时候，设置偏向锁标志为0即可）【偏向锁是为了提升在无锁情况下性能】，如果偏向锁使用过程中，发生锁竞争（有其他线程申请锁），偏向锁会升级为轻量级锁【轻量级锁相比重量级锁，可以避免线程上下文的切换，另外线程从挂起到恢复需要从用户太切换到内核态，时间成本较高】

如果在使用轻量级锁的过程中，也发生冲突，申请锁的线程会首先使用自旋的方式来尝试获取锁，如果自旋到达一定次数或者超过一定时间，锁就会升级为重量级锁，之后申请锁的线程就会进入等待状态。