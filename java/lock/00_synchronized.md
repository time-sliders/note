# synchronized

## 17.1. Synchronization 

[Synchronization](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.1) 

// TODO 十七章还有很多东西，要看完

The Java programming language provides multiple mechanisms for communicating between threads. The most basic of these methods is *synchronization*, which is implemented using *monitors*. **Each object in Java is associated with a monitor, which a thread can *lock* or *unlock***. Only one thread at a time may hold a lock on a monitor. Any other threads attempting to lock that monitor are blocked until they can obtain a lock on that monitor. A thread *t* may lock a particular monitor multiple times; each unlock reverses the effect of one lock operation.

The `synchronized` statement ([§14.19](https://docs.oracle.com/javase/specs/jls/se12/html/jls-14.html#jls-14.19)) computes a reference to an object; it then attempts to perform a lock action on that object's monitor and does not proceed further until the lock action has successfully completed. After the lock action has been performed, the body of the `synchronized` statement is executed. If execution of the body is ever completed, either normally or abruptly, an unlock action is automatically performed on that same monitor.

A `synchronized` method ([§8.4.3.6](https://docs.oracle.com/javase/specs/jls/se12/html/jls-8.html#jls-8.4.3.6)) automatically performs a lock action when it is invoked; its body is not executed until the lock action has successfully completed. If the method is an instance method, it locks the monitor associated with the instance for which it was invoked (that is, the object that will be known as `this` during execution of the body of the method). If the method is `static`, it locks the monitor associated with the `Class` object that represents the class in which the method is defined. If execution of the method's body is ever completed, either normally or abruptly, an unlock action is automatically performed on that same monitor.

The Java programming language neither prevents nor requires detection of deadlock conditions. Programs where threads hold (directly or indirectly) locks on multiple objects should use conventional techniques for deadlock avoidance, creating higher-level locking primitives that do not deadlock, if necessary.

Other mechanisms, such as reads and writes of `volatile` variables and the use of classes in the `java.util.concurrent` package, provide alternative ways of synchronization.

## 14.19. The `synchronized` Statement

[The `synchronized` Statement](https://docs.oracle.com/javase/specs/jls/se12/html/jls-14.html#jls-14.19)

A `synchronized` statement acquires a mutual-exclusion lock ([§17.1](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.1)) on behalf of the executing thread, executes a block, then releases the lock. While the executing thread owns the lock, no other thread may acquire the lock.

SynchronizedStatement:

```java
synchronized ( Expression ) Block
```

**The type of *Expression* must be a reference type, or a compile-time error occurs.**

A `synchronized` statement is executed by first evaluating the *Expression*. Then:

- If evaluation of the *Expression* completes abruptly for some reason, then the `synchronized` statement completes abruptly for the same reason.
- **Otherwise, if the value of the *Expression* is `null`, a `NullPointerException` is thrown.**
- Otherwise, let the non-`null` value of the *Expression* be `V`. The executing thread locks the monitor associated with `V`. Then the *Block* is executed, and then there is a choice:
  - If execution of the *Block* completes normally, then the monitor is unlocked and the `synchronized` statement completes normally.
  - If execution of the *Block* completes abruptly for any reason, then the monitor is unlocked and the `synchronized` statement completes abruptly for the same reason.

The locks acquired by `synchronized` statements are the same as the locks that are acquired implicitly by `synchronized` methods ([§8.4.3.6](https://docs.oracle.com/javase/specs/jls/se12/html/jls-8.html#jls-8.4.3.6)). A single thread may acquire a lock more than once.

Acquiring the lock associated with an object does not in itself prevent other threads from accessing fields of the object or invoking un-`synchronized` methods on the object. Other threads can also use `synchronized` methods or the `synchronized` statement in a conventional manner to achieve mutual exclusion.

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

A `synchronized` method acquires a monitor ([§17.1](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.1)) before it executes.

For a class (`static`) method, the monitor associated with the `Class` object for the method's class is used.

For an instance method, the monitor associated with `this` (the object for which the method was invoked) is used.



**Example 8.4.3.6-1. synchronized Monitors**

These are the same monitors that can be used by the `synchronized` statement ([§14.19](https://docs.oracle.com/javase/specs/jls/se12/html/jls-14.html#jls-14.19)).

Thus, the code:

```
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

```
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

```
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