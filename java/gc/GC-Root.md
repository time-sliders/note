# Garbage Collection Roots

<u>A garbage collection root is an object that is accessible`可访问` from **outside the heap**</u>. The following reasons make an object a GC root:

- **System Class**
  **Class loaded by bootstrap/system class loader**. For example, everything from the rt.jar like java.util.\* .
  
  Class - 由系统类加载器(system class loader)加载的对象，这些类是不能够被回收的，他们可以以静态字段的方式保存持有其它对象。我们需要注意的一点就是，通过用户自定义的类加载器加载的类，除非相应的java.lang.Class实例以其它的某种（或多种）方式成为roots，否则它们并不是roots。
  
- Thread
  **A started, but not stopped, thread. **活着的线程
  
- Thread Block
  **Object referred to from a currently active thread block.**
  
- Busy Monitor
  **Everything that has called wait() or notify() or that is synchronized.** For example, by calling synchronized(Object) or by entering a synchronized method. Static method means class, non-static method means object.
  
- Java Local
  Local variable. For example, input parameters or locally created objects of methods that are still in the stack of a thread.
  
- JNI Local
  Local variable in native code, such as user defined JNI code or JVM internal code.
  
- JNI Global
  Global variable in native code, such as user defined JNI code or JVM internal code.
  
- Native Stack
  In or out parameters in native code, such as user defined JNI code or JVM internal code. This is often the case as many methods have native parts and the objects handled as method parameters become GC roots. For example, parameters used for file/network I/O methods or reflection.
  
- Finalizable
  An object which is in a queue awaiting its finalizer to be run.
  
- Unfinalized
  An object which has a finalize method, but has not been finalized and is not yet on the finalizer queue.
  
- Unreachable
  An object which is unreachable from any other root, but has been marked as a root by MAT to retain objects which otherwise would not be included in the analysis.
  
- Java Stack Frame
  A Java stack frame, holding local variables. Only generated when the dump is parsed with the preference set to treat Java stack frames as objects.
  
- Unknown
  An object of unknown root type. Some dumps, such as IBM Portable Heap Dump files, do not have root information. For these dumps the MAT parser marks objects which are have no inbound references or are unreachable from any other root as roots of this type. This ensures that MAT retains all the objects in the dump.

---

**GC管理的主要区域是Java堆，一般情况下只针对堆进行垃圾回收。方法区、栈和本地方法区不被GC所管理,因而选择这些区域内的对象作为GC roots, 被GC roots引用的对象不被GC回收。**

Tracing GC 的根本思路就是：**给定一个集合的引用作为根出发，通过引用关系遍历对象图，能被遍历到的（可到达的）对象就被判定为存活，其余对象（也就是没有被遍历到的）就自然被判定为死亡。注意再注意：tracing GC的本质是通过找出所有活对象来把其余空间认定为“无用”，而不是找出所有死掉的对象并回收它们占用的空间。**

GC roots 这组引用是 tracing GC 的**起点**。要实现语义正确的 tracing GC，就必须要能完整枚举出**所有的GC roots**，否则就可能会漏扫描应该存活的对象，导致GC错误回收了这些被漏扫的活对象。

那么分代式GC 对 GC roots 的定义有什么影响呢？
答案是：**分代式GC是一种部分收集（partial collection）的做法。在执行部分收集时，从GC堆的非收集部分指向收集部分的引用，也必须作为GC roots的一部分**

换句话说，minor GC 比 major GC 的 GC roots 还要更大一些。如果不能理解这个道理，那整个讨论也就无从谈起了