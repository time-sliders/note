# synchronized

## 17.1. Synchronization 

[Synchronization](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.1) 

The Java programming language provides multiple mechanisms for communicating between threads. The most basic of these methods is *synchronization*, which is implemented using ***monitors***. **Each object in Java is associated with a monitor, which a thread can *lock* or *unlock***. Only one thread at a time may hold a lock on a monitor. Any other threads attempting to lock that monitor are blocked until they can obtain a lock on that monitor. A thread *t* may lock a particular monitor multiple times; each unlock reverses the effect of one lock operation.

The `synchronized` statement computes a **reference to an object**; it then attempts to perform a lock action **on that object's monitor** and does not proceed further`进一步执行` until the lock action has successfully completed. After the lock action has been performed, the body of the `synchronized` statement is executed. If execution of the body is ever completed, either normally`正常地` or abruptly`突然地`, an `unlock` action is automatically performed on that same monitor.

A `synchronized` method automatically **performs a lock action when it is invoked**; its body is not executed until the lock action has successfully completed. **If the method is an instance method, it locks the monitor associated with the instance for which it was invoked** (that is, the object that will be known as `this` during execution of the body of the method). **If the method is `static`, it locks the monitor associated with the `Class` object that represents the class in which the method is defined**. If execution of the method's body is ever completed, either normally or abruptly, an unlock action is automatically performed on that same monitor.

The Java programming language neither prevents nor requires detection`检测` of deadlock conditions. Programs where threads hold (directly or indirectly) locks on multiple objects should use conventional techniques for deadlock avoidance`避免死锁的常规`, creating higher-level locking primitives`原语` that do not deadlock, if necessary.

Other mechanisms, such as reads and writes of `volatile` variables and the use of classes in the `java.util.concurrent` package, provide alternative`可供替代的` ways of synchronization.

## 17.2. Wait Sets and Notification

Every object, in addition to having an associated monitor, has an associated *wait set*. **A wait set is a set of threads**.

When an object is first created, its wait set is empty. Elementary actions`基础操作` that add threads to and remove threads from wait sets are atomic`原子性的`. Wait sets are manipulated`控制` solely`单独` through the methods`Object.wait`, `Object.notify`, and `Object.notifyAll`.

**Wait set manipulations can also be affected by the interruption status of a thread**, and by the `Thread` class's methods dealing with interruption. Additionally`另外`, the `Thread` class's methods for sleeping and joining other threads have properties derived`来源` from those of wait and notification actions.

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