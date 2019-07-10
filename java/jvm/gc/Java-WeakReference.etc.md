# 1 Java引用介绍

Java从1.2版本开始引入了4种引用，这4种引用的级别由高到低依次为：

**强引用  >  软引用  >  弱引用  >  虚引用**

## StrongReference
强引用是使用最普遍的引用。如果一个对象具有强引用，那垃圾回收器绝不会回收它。当内存空间不足，Java虚拟机宁愿抛出OutOfMemoryError错误，使程序异常终止，也不会靠随意回收具有强引用的对象来解决内存不足的问题。

如 Date date = new Date();   此时 date 即是一个对内存中的 new Date() 对象的强引用。

## SoftReference  软引用

Soft reference objects, which are **cleared** *at the discretion of*由…决定 the garbage collector in response to memory demand强烈要求. Soft references are most often used to implement *memory-sensitive caches*内存敏感的缓存.
软引用对象，会在垃圾收集器强烈需要内存的情况下被清除。软引用经常被用于实现内存敏感的缓存。

Suppose that the garbage collector determines at a certain point in time that an object is softly reachable. At that time it **may choose** to clear atomically all soft references to that object and all soft references to any other softly-reachable objects from which that object is reachable through a chain of strong references. At the same time or at some later time** it will enqueue those newly-cleared soft references that are registered with reference queues.
如果垃圾收集器确定在某个时间点一个对象可以softly reachable。此时，它可以选择原子性的清除到这个对象的所有软引用，以及所有从这个对象的强引用链可达的其他softly-reachable对象的软引用。在同一时间或者稍后，它会将这些刚刚清除的注册的软引用入队到一个reference queues。

All soft references to softly-reachable objects are guaranteed to have been cleared before the virtual machine throws an OutOfMemoryError. Otherwise no constraints are placed upon the time at which a soft reference will be cleared or the order in which a set of such references to different objects will be cleared. Virtual machine implementations are, however, **encouraged to bias against clearing recently-created or recently-used soft references.**
所有软引用指向 softly-reachable对象，可以保证在虚拟机抛出OutOfMemoryError之前被清除。另外不会有对在什么时候软引用会被移除、以及对一组不同对象的软引用移除的顺序的约束。然而，根据不同虚拟机的实现，鼓励偏向于反对清除最近创建的或者最近使用的软引用。

Direct instances of this class may be used to implement simple caches; this class or derived subclasses may also be used in larger data structures to implement more sophisticated caches. As long as the referent of a soft reference is strongly reachable, that is, is actually in use, the soft reference will not be cleared. Thus a sophisticated cache can, for example, prevent its most recently used entries from being discarded by keeping strong referents to those entries, leaving the remaining entries to be discarded at the discretion of the garbage collector
该类的直接实例可以用于实现简单缓存；该类或者子类也可以供较大的数据结构来实现更加复杂的缓存。只要软引用的引用是强可到达的，也就是说，实际上正在使用中，软引用就不会被清除。因此，例如，一个复杂的缓存可以通过保持对这些条目的强引用来防止其最近使用的条目被丢弃，剩下的条目将由垃圾收集器自行丢弃。

## WeakReference 弱引用

Weak reference objects, which do not prevent their referents from being made finalizable, finalized, and then reclaimed. Weak references are most often used to implement canonicalizing mappings.
弱引用对象，不会阻止他的引用被标记为finalizable，finalized，然后被回收。弱引用经常用来实现标准映射。

Suppose that the garbage collector determines at a certain point in time that an object is weakly reachable. At that time it **will** atomically clear all weak references to that object and all weak references to any other weakly-reachable objects from which that object is reachable through a chain of strong and soft references. **At the same time** it will declare all of the formerly weakly-reachable objects to be finalizable. At the same time or at some later time it will enqueue those newly-cleared weak references that are registered with reference queues.
如果垃圾收集器检测在某个时间点一个对象是weakly-reachable，此时，它就会原子性的清除到这个对象的所有弱引用，以及所有从这个对象的强引用链或者软引用链可达的其他weakly-reachable对象的弱引用。在同时，它会申明所有之前的weakly-reachable对象为finalizable，它会将这些刚刚清除的注册的软引用入队到一个reference queues。在同一时间或者稍后，它会将这些刚刚清除的注册的软引用入队到一个reference queues。



## PhantomReference 虚引用

Phantom reference objects, which are enqueued after the collector determines that their referents may otherwise be reclaimed. Phantom references are most often used for scheduling pre-mortem cleanup actions in a more flexible way than is possible with the Java finalization mechanism.
虚引用对象，在垃圾收集器检测他的引用可能被回收之后入队。虚引用常用来以比Java finalization 机制可能更灵活的方式调度预验尸清理动作。

If the garbage collector determines at a certain point in time that the referent of a phantom reference is phantom reachable, then at that time or at some later time it will enqueue the reference.
如果垃圾收集器确定在某一个时间点一个虚引用对象是phantom reachable，那么在此时或者一段时间之后，他就会入队这个引用。

In order to ensure that a reclaimable object remains so, the referent of a phantom reference may not be retrieved: The get method of a phantom reference always returns null.
为了确保可回收对象保持不变，可能无法检索虚引用的引用：虚引用的get方法始终返回空值。

Unlike soft and weak references, phantom references are not automatically cleared by the garbage collector as they are enqueued. An object that is reachable via phantom references will remain so until all such references are cleared or themselves become unreachable.
与软引用和弱引用不同，虚引用在入队时，不会自动被垃圾收集器清除。通过虚引用可访问的对象将保持不变，直到清除所有这些引用或它们自己变得不可访问为止。



# 测试

```java
public class MyObject {

    private String name;

    private static volatile boolean isOOM = false;

    private static volatile long start;

    private MyObject(String name) {
        this.name = name;
    }

    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        if (!isOOM) {
            System.out.println(name + " was reclaimed before oom, cost:" + (System.currentTimeMillis() - start));
        } else {
            System.out.println(name + " was reclaimed after oom, cost:" + (System.currentTimeMillis() - start));
        }
    }

    @SuppressWarnings("InfiniteLoopStatement")
    private static void fill() {
        List<Long> l = new LinkedList<>();
        try {
            for (Long i = 128L; ; i++) {
                l.add(i);
            }
        } finally {
            double size = ((4.0 + 4 + 4 + 4 + (l.size() * 4)
                    + l.size() * (4 + 4 + 4 + 8)) / 1024) / 1024;
            System.out.println(size);
        }
    }


    public static void main(String[] args) {
        start = System.currentTimeMillis();
        MyObject o = new MyObject("stronger reference");
        SoftReference<MyObject> s = new SoftReference<MyObject>(new MyObject("soft reference"));
        WeakReference<MyObject> w = new WeakReference<MyObject>(new MyObject("weak reference"));
        PhantomReference<MyObject> p = new PhantomReference<MyObject>(new MyObject("phantom reference"), new ReferenceQueue<>());
        try {
            fill();
        } finally {
            isOOM = true;
            System.out.println("total cost:" + (System.currentTimeMillis() - start));
        }
    }

    /* 程序输出
     phantom reference was reclaimed before oom, cost:134
     weak reference was reclaimed before oom, cost:134
     soft reference was reclaimed before oom, cost:843
     6.2587432861328125
     total cost:915
     stronger reference was reclaimed after oom, cost:921

     Exception in thread "main"
     java.lang.OutOfMemoryError: Java heap space
     */
}
```