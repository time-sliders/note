## 17.4. Memory Model

[§17.4](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4) 

A *memory model* describes`描述的是:`, given a **program** and an **execution trace of that program**, whether`是否` the execution trace is a legal execution of the program. The Java programming language memory model works by **examining`检查` each read in an execution trace and checking that the write observed`看到的` by that read is valid according to certain`某些` rules.**

The memory model describes possible behaviors of a program. An implementation is free to produce`生成` any code it likes, as long as`只要` all resulting executions of a program produce a result that can be predicted`预测` by the memory model.

This provides a great deal of`大量的` freedom for the implementor`实现者` to perform a myriad of`无数的` code transformations`转换`, including the reordering`重排序` of actions and removal`移除` of unnecessary synchronization.



**Example 17.4-1. Incorrectly`不正确的` Synchronized Programs May Exhibit`表现出` Surprising`令人惊讶的`  Behavior**

The semantics`语法` of the Java programming language allow compilers and microprocessors`微处理器` to perform optimizations`执行优化` that can interact with`与…交互` incorrectly synchronized code in ways that can produce behaviors that seem paradoxical`看似矛盾`. Here are some examples of how incorrectly synchronized programs may exhibit surprising behaviors.

Consider, for example, the example program traces shown in [Table 17.4-A](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4-A). This program uses local variables `r1` and `r2` and shared variables `A` and `B`. Initially, `A == B == 0`.



**Table 17.4-A. Surprising results caused by statement reordering - original code**

| Thread 1     | Thread 2     |
| ------------ | ------------ |
| 1: `r2 = A;` | 3: `r1 = B;` |
| 2: `B = 1;`  | 4: `A = 2;`  |

It may appear that the result `r2 == 2` and `r1 == 1` is impossible. Intuitively`直觉地`, either instruction`指令` 1 or`要么...或者...` instruction 3 should come first in an execution. If instruction 1 comes first, it should not be able to see the write at instruction 4. If instruction 3 comes first, it should not be able to see the write at instruction 2.

If some execution exhibited this behavior, then we would know that instruction 4 came before instruction 1, which came before instruction 2, which came before instruction 3, which came before instruction 4. This is, on the face of it, absurd`从表面上看，这是荒谬的`.

However, compilers are allowed to **reorder the instructions** in either`任何` thread, when this does not affect the execution of that thread in isolation`不会单独影响该线程的执行`. If instruction 1 is reordered with instruction 2, as shown in the trace in [Table 17.4-B](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4-B), then it is easy to see how the result `r2 == 2` and `r1 == 1` might occur.



**Table 17.4-B. Surprising results caused by statement reordering - valid compiler transformation**

| Thread 1  | Thread 2  |
| --------- | --------- |
| `B = 1;`  | `r1 = B;` |
| `r2 = A;` | `A = 2;`  |

To some programmers, this behavior may seem "broken". However, it should be noted that this code is improperly`不恰当的` synchronized:

- **there is a write in one thread,**
- **a read of the same variable by another thread,**
- **and the write and read are not ordered by synchronization.**

This situation is an example of a *data race`数据竞争`* (§17.4.5). When code contains a data race, counterintuitive`违反直觉的` results are often possible.

Several mechanisms`几种机制` can produce the **reordering** in [*Table 17.4-B*]. A Just-In-Time compiler`即时编译器` in a Java Virtual Machine implementation may rearrange`重新安排` code, or the processor. In addition`此外`, the memory hierarchy of the architecture on which a Java Virtual Machine implementation is run may make it appear as if code is being reordered`运行Java虚拟机实现的内存层次结构的架构，可以使其看起来像是被重新排序过的代码`. In this chapter, we shall refer to anything that can reorder code as a *compiler*`我们将把任何可以重新排序代码的东西称为编译器`.

Another example of surprising results can be seen in [Table 17.4-C](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4-C). Initially, `p == q` and `p.x == 0`. This program is also incorrectly synchronized; it writes to **shared memory** without enforcing any ordering between those writes.



**Table 17.4-C. Surprising results caused by forward substitution**`正向替换`

| Thread 1     | Thread 2    |
| ------------ | ----------- |
| `r1 = p;`    | `r6 = p;`   |
| `r2 = r1.x;` | `r6.x = 3;` |
| `r3 = q;`    |             |
| `r4 = r3.x;` |             |
| `r5 = r1.x;` |             |

One common compiler optimization involves having the value read for `r2` reused for `r5`: they are both reads of `r1.x` with no intervening write. This situation is shown in [Table 17.4-D](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4-D).



**Table 17.4-D. Surprising results caused by forward substitution**

| Thread 1     | Thread 2    |
| ------------ | ----------- |
| `r1 = p;`    | `r6 = p;`   |
| `r2 = r1.x;` | `r6.x = 3;` |
| `r3 = q;`    |             |
| `r4 = r3.x;` |             |
| `r5 = r2;`   |             |

Now consider the case where the assignment to `r6.x` in Thread 2 happens between the first read of `r1.x` and the read of `r3.x` in Thread 1. If the compiler decides to reuse`重用` the value of `r2` for the `r5`, then `r2` and `r5` will have the value `0`, and `r4` will have the value `3`. From the perspective`视角` of the programmer, the value stored at `p.x` has changed from `0` to `3`and then changed back.

**The memory model determines`决定` what values can be read at every point in the program**. The actions of each thread in isolation`独立的` must behave as governed`治理` by the semantics`语义` of that thread, with the exception that the values seen by each read are determined by the memory model. When we refer to`认同` this, we say that the program obeys`服从` *intra-thread semantics*`线程内语义`. Intra-thread semantics are the semantics for single-threaded programs, and allow the complete prediction of the behavior of a thread based on the values seen by read actions within the thread`允许根据线程内读取操作看到的值对线程的行为进行完整的预测`. To determine if the actions of thread *t* in an execution are legal, we simply evaluate`评估` the implementation of thread *t* as`因为` it would be performed in a single-threaded context, as defined in the rest of`剩余部分` this specification`说明`.

Each time the evaluation of thread *t* generates an **inter-thread action**, it must match the inter-thread action *a* of *t* that comes next`下一个` in program order. If *a* is a read, then further evaluation of *t* uses the value seen by *a* as determined by the memory model.

This section provides the specification of the Java programming language memory model except for issues dealing with `final` fields, which are described in [§17.5](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.5).

The memory model specified herein`本文` is not fundamentally`从根本上` based in the object-oriented nature`面向对象本质` of the Java programming language. For conciseness and simplicity`为了简洁起见` in our examples, we often exhibit code fragments`片段` without class or method definitions, or explicit`隐式` dereferencing. Most examples consist of two or more threads containing statements with access to local variables, shared global variables, or instance fields of an object. We typically use variables names such as `r1` or `r2` to indicate variables local to a method or thread. Such variables are not accessible by other threads.

### 17.4.1. Shared Variables

**Memory that can be shared between threads is called *shared memory* or *heap memory*.**

All instance fields, `static` fields, and array elements are stored in heap memory. In this chapter, we use the term`把…称为` *variable* to refer to both fields and array elements.

Local variables ([§14.4](https://docs.oracle.com/javase/specs/jls/se12/html/jls-14.html#jls-14.4)), formal method parameters`方法行参` ([§8.4.1](https://docs.oracle.com/javase/specs/jls/se12/html/jls-8.html#jls-8.4.1)), and exception handler parameters ([§14.20](https://docs.oracle.com/javase/specs/jls/se12/html/jls-14.html#jls-14.20)) are never shared between threads and are unaffected by the memory model.

Two accesses to (reads of or writes to) the same variable are said to be *conflicting* if at least one of the accesses is a write.

### 17.4.2. Actions

An *inter-thread action* is an action performed by one thread that can be detected or directly influenced`影响` by another thread. There are several kinds of inter-thread action that a program may perform:

- *Read* (normal, or non-volatile`长存`). Reading a variable.

- *Write* (normal, or non-volatile). Writing a variable.

- *Synchronization actions*, which are:

  - *Volatile read*`可变读取`. A volatile read of a variable.
  - *Volatile write*. A volatile write of a variable.
  - *Lock*. Locking a monitor
  - *Unlock*. Unlocking a monitor.
  - The (synthetic`虚拟的,合成的`) first and last action of a thread.
  - Actions that start a thread or detect that a thread has terminated ([§17.4.4](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.4)).

- *External Actions*. An external action is an action that may be observable`可观察到` outside of an execution, and has a result based on an environment external to the execution.

- *Thread divergence`分散` actions* ([§17.4.9](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.9)). A thread divergence action is only performed by a thread that is in an infinite loop in which no memory, synchronization, or external actions are performed. If a thread performs a thread divergence action, it will be followed by an infinite number of thread divergence actions.

  Thread divergence actions are introduced to model how a thread may cause all other threads to stall and fail to make progress.

This specification is only concerned with`涉及` inter-thread actions. We do not need to concern`关心` ourselves with intra-thread actions (e.g., adding two local variables and storing the result in a third local variable). As previously mentioned`如前所述`, all threads need to obey`服从` the correct intra-thread semantics for Java programs. We will usually refer to inter-thread actions more succinctly`简洁地` as simply *actions*.

An action *a* is described by a tuple < *t*, *k*, *v*, *u* >, comprising`包括`:

- *t* - the thread performing the action

- *k* - the kind of action

- *v* - the variable or monitor involved in the action.

  For lock actions, *v* is the monitor being locked; for unlock actions, *v* is the monitor being unlocked.

  If the action is a (volatile or non-volatile) read, *v* is the variable being read.

  If the action is a (volatile or non-volatile) write, *v* is the variable being written.

- *u* - an arbitrary`任意的` unique identifier for the action

An external`外部的` action tuple contains an additional component, which contains the results of the external action as perceived`感知` by the thread performing the action. This may be information as to the success or failure of the action, and any values read by the action.

Parameters to the external action (e.g., which bytes are written to which socket) are not part of the external action tuple. These parameters are set up by other actions within the thread and can be determined by examining the intra-thread semantics. They are not explicitly discussed in the memory model.

In non-terminating executions, not all external actions are observable. Non-terminating executions and observable actions are discussed in [§17.4.9](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.9).

### 17.4.3. Programs and Program Order

**Among all the inter-thread actions performed by each thread *t*, the *program order* of *t* is a total order that reflects the order in which these actions would be performed according to the intra-thread semantics of *t***.

A set of actions is *sequentially consistent`顺序一致`* if all actions occur in a total order (the execution order) that is consistent with`与…一致` program order, and furthermore`此外`, each read *r* of a variable *v* sees the value written by the write *w* to *v* such that:

- *w* comes before *r* in the execution order, and
- there is no other write *w*' such that *w* comes before *w*' and *w*' comes before *r* in the execution order.

**Sequential consistency** is a very strong guarantee that is made about visibility and ordering in an execution of a program. Within a sequentially consistent execution, there is a total order over all individual`单独的` actions (such as reads and writes) which is consistent with the order of the program, and each individual action is atomic and is immediately visible to every thread.

If a program has no data races, then all executions of the program will appear to be sequentially consistent.

Sequential consistency and/or freedom from data races still allows errors arising from groups of operations that need to be perceived atomically and are not.`顺序一致性和/或不受数据竞争的影响，仍然允许由需要原子地感知而不是原子地感知的操作组引起的错误`

If we were to use sequential consistency as our memory model, many of the compiler and processor optimizations that we have discussed would be illegal`非法`. For example, in the trace in [Table 17.4-C](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4-C), as soon as the write of `3` to `p.x` occurred, subsequent reads of that location would be required to see that value.

### 17.4.4. Synchronization Order

Every execution has a *synchronization order*. A synchronization order is a total order over all of the synchronization actions of an execution. For each thread *t*, the synchronization order of the synchronization actions ([§17.4.2](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.2)) in *t* is consistent with the program order ([§17.4.3](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.3)) of *t*.

Synchronization actions induce the *synchronized-with* relation on actions, defined as follows:

- An unlock action on monitor *m* *synchronizes-with* all subsequent lock actions on *m* (where "subsequent" is defined according to the synchronization order).

- A write to a volatile variable *v* ([§8.3.1.4](https://docs.oracle.com/javase/specs/jls/se12/html/jls-8.html#jls-8.3.1.4)) *synchronizes-with* all subsequent reads of *v* by any thread (where "subsequent" is defined according to the synchronization order).

- An action that starts a thread *synchronizes-with* the first action in the thread it starts.

- The write of the default value (zero, `false`, or `null`) to each variable *synchronizes-with* the first action in every thread.

  Although it may seem a little strange to write a default value to a variable before the object containing the variable is allocated, conceptually every object is created at the start of the program with its default initialized values.

- The final action in a thread `T1` *synchronizes-with* any action in another thread `T2` that detects that `T1` has terminated.

  `T2` may accomplish this by calling `T1.isAlive()` or `T1.join()`.

- If thread `T1` interrupts thread `T2`, the interrupt by `T1` *synchronizes-with* any point where any other thread (including `T2`) determines that `T2` has been interrupted (by having an `InterruptedException` thrown or by invoking `Thread.interrupted` or `Thread.isInterrupted`).

The source of a *synchronizes-with* edge is called a *release*, and the destination is called an *acquire*.

### 17.4.5. Happens-before Order

Two actions can be ordered by a *happens-before* relationship. If one action *happens-before* another, then the first is visible to and ordered before the second.

If we have two actions *x* and *y*, we write *hb(x, y)* to indicate that *x happens-before y*.

- If *x* and *y* are actions of the same thread and *x* comes before *y* in program order, then *hb(x, y)*.
- There is a *happens-before* edge from the end of a constructor of an object to the start of a finalizer ([§12.6](https://docs.oracle.com/javase/specs/jls/se12/html/jls-12.html#jls-12.6)) for that object.
- If an action *x* *synchronizes-with* a following action *y*, then we also have *hb(x, y)*.
- If *hb(x, y)* and *hb(y, z)*, then *hb(x, z)*.

The `wait` methods of class `Object` ([§17.2.1](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.2.1)) have lock and unlock actions associated with them; their *happens-before* relationships are defined by these associated actions.

It should be noted that the presence of a *happens-before* relationship between two actions does not necessarily imply that they have to take place in that order in an implementation. If the reordering produces results consistent with a legal execution, it is not illegal.

For example, the write of a default value to every field of an object constructed by a thread need not happen before the beginning of that thread, as long as no read ever observes that fact.

More specifically, if two actions share a *happens-before* relationship, they do not necessarily have to appear to have happened in that order to any code with which they do not share a *happens-before* relationship. Writes in one thread that are in a data race with reads in another thread may, for example, appear to occur out of order to those reads.

The *happens-before* relation defines when data races take place.

A set of synchronization edges, *S*, is *sufficient* if it is the minimal set such that the transitive closure of *S* with the program order determines all of the *happens-before* edges in the execution. This set is unique.

It follows from the above definitions that:

- An unlock on a monitor *happens-before* every subsequent lock on that monitor.
- **A write to a `volatile` field ([§8.3.1.4](https://docs.oracle.com/javase/specs/jls/se12/html/jls-8.html#jls-8.3.1.4)) *happens-before* every subsequent read of that field.**
- A call to `start()` on a thread *happens-before* any actions in the started thread.
- All actions in a thread *happen-before* any other thread successfully returns from a `join()` on that thread.
- The default initialization of any object *happens-before* any other actions (other than default-writes) of a program.

When a program contains two conflicting accesses ([§17.4.1](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.1)) that are not ordered by a happens-before relationship, it is said to contain a *data race*.

The semantics of operations other than inter-thread actions, such as reads of array lengths ([§10.7](https://docs.oracle.com/javase/specs/jls/se12/html/jls-10.html#jls-10.7)), executions of checked casts ([§5.5](https://docs.oracle.com/javase/specs/jls/se12/html/jls-5.html#jls-5.5), [§15.16](https://docs.oracle.com/javase/specs/jls/se12/html/jls-15.html#jls-15.16)), and invocations of virtual methods ([§15.12](https://docs.oracle.com/javase/specs/jls/se12/html/jls-15.html#jls-15.12)), are not directly affected by data races.

Therefore, a data race cannot cause incorrect behavior such as returning the wrong length for an array.

A program is *correctly synchronized* if and only if all sequentially consistent executions are free of data races.

If a program is correctly synchronized, then all executions of the program will appear to be sequentially consistent ([§17.4.3](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.3)).

This is an extremely strong guarantee for programmers. Programmers do not need to reason about reorderings to determine that their code contains data races. Therefore they do not need to reason about reorderings when determining whether their code is correctly synchronized. Once the determination that the code is correctly synchronized is made, the programmer does not need to worry that reorderings will affect his or her code.

A program must be correctly synchronized to avoid the kinds of counterintuitive behaviors that can be observed when code is reordered. The use of correct synchronization does not ensure that the overall behavior of a program is correct. However, its use does allow a programmer to reason about the possible behaviors of a program in a simple way; the behavior of a correctly synchronized program is much less dependent on possible reorderings. Without correct synchronization, very strange, confusing and counterintuitive behaviors are possible.

We say that a read *r* of a variable *v* is allowed to observe a write *w* to *v* if, in the *happens-before* partial order of the execution trace:

- *r* is not ordered before *w* (i.e., it is not the case that *hb(r, w)*), and
- there is no intervening write *w*' to *v* (i.e. no write *w*' to *v* such that *hb(w, w')* and *hb(w', r)*).

Informally, a read *r* is allowed to see the result of a write *w* if there is no *happens-before* ordering to prevent that read.

A set of actions *A* is *happens-before consistent* if for all reads *r* in *A*, where *W(r)* is the write action seen by *r*, it is not the case that either *hb(r, W(r))* or that there exists a write *w* in *A* such that *w.v* = *r.v* and *hb(W(r), w)* and *hb(w, r)*.

In a *happens-before consistent* set of actions, each read sees a write that it is allowed to see by the *happens-before* ordering.



**Example 17.4.5-1. Happens-before Consistency**

For the trace in [Table 17.4.5-A](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.5-A), initially `A == B == 0`. The trace can observe `r2 == 0` and `r1 == 0` and still be *happens-before consistent*, since there are execution orders that allow each read to see the appropriate write.



**Table 17.4.5-A. Behavior allowed by happens-before consistency, but not sequential consistency.**

| Thread 1  | Thread 2  |
| --------- | --------- |
| `B = 1;`  | `A = 2;`  |
| `r2 = A;` | `r1 = B;` |

Since there is no synchronization, each read can see either the write of the initial value or the write by the other thread. An execution order that displays this behavior is:

```
1: B = 1;
3: A = 2;
2: r2 = A;  // sees initial write of 0
4: r1 = B;  // sees initial write of 0
```

Another execution order that is happens-before consistent is:

```
1: r2 = A;  // sees write of A = 2
3: r1 = B;  // sees write of B = 1
2: B = 1;
4: A = 2;
```

In this execution, the reads see writes that occur later in the execution order. This may seem counterintuitive, but is allowed by *happens-before* consistency. Allowing reads to see later writes can sometimes produce unacceptable behaviors.

### 17.4.6. Executions

An execution *E* is described by a tuple < *P, A, po, so, W, V, sw, hb* >, comprising:

- *P* - a program
- *A* - a set of actions
- *po* - program order, which for each thread *t*, is a total order over all actions performed by *t* in *A*
- *so* - synchronization order, which is a total order over all synchronization actions in *A*
- *W* - a write-seen function, which for each read *r* in *A*, gives *W(r)*, the write action seen by *r* in *E*.
- *V* - a value-written function, which for each write *w* in *A*, gives *V(w)*, the value written by *w* in *E*.
- *sw* - synchronizes-with, a partial order over synchronization actions
- *hb* - happens-before, a partial order over actions

Note that the synchronizes-with and happens-before elements are uniquely determined by the other components of an execution and the rules for well-formed executions ([§17.4.7](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.7)).

An execution is *happens-before consistent* if its set of actions is *happens-before consistent* ([§17.4.5](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.5)).

### 17.4.7. Well-Formed Executions

We only consider well-formed executions. An execution *E* = < *P, A, po, so, W, V, sw, hb* > is well formed if the following are true:

1. Each read sees a write to the same variable in the execution.

   All reads and writes of volatile variables are volatile actions. For all reads *r* in *A*, we have *W(r)* in *A* and *W(r).v* = *r.v*. The variable *r.v* is volatile if and only if *r* is a volatile read, and the variable *w.v*is volatile if and only if *w* is a volatile write.

2. The happens-before order is a partial order.

   The happens-before order is given by the transitive closure of synchronizes-with edges and program order. It must be a valid partial order: reflexive, transitive and antisymmetric.

3. The execution obeys intra-thread consistency.

   For each thread *t*, the actions performed by *t* in *A* are the same as would be generated by that thread in program-order in isolation, with each write *w* writing the value *V(w)*, given that each read *r*sees the value *V(W(r))*. Values seen by each read are determined by the memory model. The program order given must reflect the program order in which the actions would be performed according to the intra-thread semantics of *P*.

4. The execution is *happens-before consistent* ([§17.4.6](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.6)).

5. The execution obeys synchronization-order consistency.

   For all volatile reads *r* in *A*, it is not the case that either *so(r, W(r))* or that there exists a write *w* in *A* such that *w.v* = *r.v* and *so(W(r), w)* and *so(w, r)*.

### 17.4.8. Executions and Causality Requirements

We use *f*|*d* to denote the function given by restricting the domain of *f* to *d*. For all *x* in *d*, *f*|*d*(*x*) = *f*(*x*), and for all *x* not in *d*, *f*|*d*(*x*) is undefined.

We use *p*|*d* to represent the restriction of the partial order *p* to the elements in *d*. For all *x*,*y* in *d*, *p*(*x*,*y*) if and only if *p*|*d*(*x*,*y*). If either *x* or *y* are not in *d*, then it is not the case that *p*|*d*(*x*,*y*).

A well-formed execution *E* = < *P, A, po, so, W, V, sw, hb* > is validated by *committing* actions from *A*. If all of the actions in *A* can be committed, then the execution satisfies the causality requirements of the Java programming language memory model.

Starting with the empty set as *C0*, we perform a sequence of steps where we take actions from the set of actions *A* and add them to a set of committed actions *Ci* to get a new set of committed actions *Ci+1*. To demonstrate that this is reasonable, for each *Ci* we need to demonstrate an execution *E* containing *Ci* that meets certain conditions.

Formally, an execution *E satisfies the causality requirements of the Java programming language memory model* if and only if there exist:

- Sets of actions *C0*, *C1*, ... such that:

  - *C0* is the empty set
  - *Ci* is a proper subset of *Ci+1*
  - *A* = ∪ (*C0*, *C1*, ...)

  If *A* is finite, then the sequence *C0*, *C1*, ... will be finite, ending in a set *Cn* = *A*.

  If *A* is infinite, then the sequence *C0*, *C1*, ... may be infinite, and it must be the case that the union of all elements of this infinite sequence is equal to *A*.

- Well-formed executions *E1*, ..., where *Ei* = < *P, Ai, poi, soi, Wi, Vi, swi, hbi* >.

Given these sets of actions *C0*, ... and executions *E1*, ... , every action in *Ci* must be one of the actions in *Ei*. All actions in *Ci* must share the same relative happens-before order and synchronization order in both *Ei* and *E*. Formally:

1. *Ci* is a subset of *Ai*
2. *hbi*|*Ci* = *hb*|*Ci*
3. *soi*|*Ci* = *so*|*Ci*

The values written by the writes in *Ci* must be the same in both *Ei* and *E*. Only the reads in *Ci-1* need to see the same writes in *Ei* as in *E*. Formally:

1. *Vi*|*Ci* = *V*|*Ci*
2. *Wi*|*Ci-1* = *W*|*Ci-1*

All reads in *Ei* that are not in *Ci-1* must see writes that happen-before them. Each read *r* in *Ci* - *Ci-1* must see writes in *Ci-1* in both *Ei* and *E*, but may see a different write in *Ei* from the one it sees in*E*. Formally:

1. For any read *r* in *Ai* - *Ci-1*, we have *hbi(Wi(r), r)*
2. For any read *r* in (*Ci* - *Ci-1*), we have *Wi(r)* in *Ci-1* and *W(r)* in *Ci-1*

Given a set of sufficient synchronizes-with edges for *Ei*, if there is a release-acquire pair that happens-before ([§17.4.5](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.5)) an action you are committing, then that pair must be present in all *Ej*, where *j* ≥*i*. Formally:

1. Let *sswi* be the *swi* edges that are also in the transitive reduction of *hbi* but not in *po*. We call *sswi* the *sufficient synchronizes-with edges for Ei*. If *sswi(x, y)* and *hbi(y, z)* and *z* in *Ci*, then *swj(x, y)*for all *j* ≥ *i*.

   If an action *y* is committed, all external actions that happen-before *y* are also committed.

2. If *y* is in *Ci*, *x* is an external action and *hbi(x, y)*, then *x* in *Ci*.



**Example 17.4.8-1. Happens-before Consistency Is Not Sufficient**

Happens-before consistency is a necessary, but not sufficient, set of constraints. Merely enforcing happens-before consistency would allow for unacceptable behaviors - those that violate the requirements we have established for programs. For example, happens-before consistency allows values to appear "out of thin air". This can be seen by a detailed examination of the trace in [Table 17.4.8-A](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.8-A).



**Table 17.4.8-A. Happens-before consistency is not sufficient**

| Thread 1              | Thread 2              |
| --------------------- | --------------------- |
| `r1 = x;`             | `r2 = y;`             |
| `if (r1 != 0) y = 1;` | `if (r2 != 0) x = 1;` |

The code shown in [Table 17.4.8-A](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.8-A) is correctly synchronized. This may seem surprising, since it does not perform any synchronization actions. Remember, however, that a program is correctly synchronized if, when it is executed in a sequentially consistent manner, there are no data races. If this code is executed in a sequentially consistent way, each action will occur in program order, and neither of the writes will occur. Since no writes occur, there can be no data races: the program is correctly synchronized.

Since this program is correctly synchronized, the only behaviors we can allow are sequentially consistent behaviors. However, there is an execution of this program that is happens-before consistent, but not sequentially consistent:

```
r1 = x;  // sees write of x = 1
y = 1;
r2 = y;  // sees write of y = 1
x = 1; 
```

This result is happens-before consistent: there is no happens-before relationship that prevents it from occurring. However, it is clearly not acceptable: there is no sequentially consistent execution that would result in this behavior. The fact that we allow a read to see a write that comes later in the execution order can sometimes thus result in unacceptable behaviors.

Although allowing reads to see writes that come later in the execution order is sometimes undesirable, it is also sometimes necessary. As we saw above, the trace in [Table 17.4.5-A](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.5-A)requires some reads to see writes that occur later in the execution order. Since the reads come first in each thread, the very first action in the execution order must be a read. If that read cannot see a write that occurs later, then it cannot see any value other than the initial value for the variable it reads. This is clearly not reflective of all behaviors.

We refer to the issue of when reads can see future writes as *causality*, because of issues that arise in cases like the one found in [Table 17.4.8-A](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.8-A). In that case, the reads cause the writes to occur, and the writes cause the reads to occur. There is no "first cause" for the actions. Our memory model therefore needs a consistent way of determining which reads can see writes early.

Examples such as the one found in [Table 17.4.8-A](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.8-A) demonstrate that the specification must be careful when stating whether a read can see a write that occurs later in the execution (bearing in mind that if a read sees a write that occurs later in the execution, it represents the fact that the write is actually performed early).

The memory model takes as input a given execution, and a program, and determines whether that execution is a legal execution of the program. It does this by gradually building a set of "committed" actions that reflect which actions were executed by the program. Usually, the next action to be committed will reflect the next action that can be performed by a sequentially consistent execution. However, to reflect reads that need to see later writes, we allow some actions to be committed earlier than other actions that happen-before them.

Obviously, some actions may be committed early and some may not. If, for example, one of the writes in [Table 17.4.8-A](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.8-A) were committed before the read of that variable, the read could see the write, and the "out-of-thin-air" result could occur. Informally, we allow an action to be committed early if we know that the action can occur without assuming some data race occurs. In [Table 17.4.8-A](https://docs.oracle.com/javase/specs/jls/se12/html/jls-17.html#jls-17.4.8-A), we cannot perform either write early, because the writes cannot occur unless the reads see the result of a data race.

### 17.4.9. Observable Behavior and Nonterminating Executions

For programs that always terminate in some bounded finite period of time, their behavior can be understood (informally) simply in terms of their allowable executions. For programs that can fail to terminate in a bounded amount of time, more subtle issues arise.

The observable behavior of a program is defined by the finite sets of external actions that the program may perform. A program that, for example, simply prints "Hello" forever is described by a set of behaviors that for any non-negative integer *i*, includes the behavior of printing "Hello" *i* times.

Termination is not explicitly modeled as a behavior, but a program can easily be extended to generate an additional external action *executionTermination* that occurs when all threads have terminated.

We also define a special *hang* action. If behavior is described by a set of external actions including a *hang* action, it indicates a behavior where after the external actions are observed, the program can run for an unbounded amount of time without performing any additional external actions or terminating. Programs can hang if all threads are blocked or if the program can perform an unbounded number of actions without performing any external actions.

A thread can be blocked in a variety of circumstances, such as when it is attempting to acquire a lock or perform an external action (such as a read) that depends on external data.

An execution may result in a thread being blocked indefinitely and the execution's not terminating. In such cases, the actions generated by the blocked thread must consist of all actions generated by that thread up to and including the action that caused the thread to be blocked, and no actions that would be generated by the thread after that action.

To reason about observable behaviors, we need to talk about sets of observable actions.

If *O* is a set of observable actions for an execution *E*, then set *O* must be a subset of *E*'s actions, *A*, and must contain only a finite number of actions, even if *A* contains an infinite number of actions. Furthermore, if an action *y* is in *O*, and either *hb(x, y)* or *so(x, y)*, then *x* is in *O*.

Note that a set of observable actions are not restricted to external actions. Rather, only external actions that are in a set of observable actions are deemed to be observable external actions.

A behavior *B* is an allowable behavior of a program *P* if and only if *B* is a finite set of external actions and either:

- There exists an execution *E* of *P*, and a set *O* of observable actions for *E*, and *B* is the set of external actions in *O* (If any threads in *E* end in a blocked state and *O* contains all actions in *E*, then *B*may also contain a *hang* action); or
- There exists a set *O* of actions such that *B* consists of a *hang* action plus all the external actions in *O* and for all *k* ≥ | *O* |, there exists an execution *E* of *P* with actions *A*, and there exists a set of actions *O*' such that:
  - Both *O* and *O*' are subsets of *A* that fulfill the requirements for sets of observable actions.
  - *O* ⊆ *O*' ⊆ *A*
  - | *O*' | ≥ *k*
  - *O*' - *O* contains no external actions

Note that a behavior *B* does not describe the order in which the external actions in *B* are observed, but other (internal) constraints on how the external actions are generated and performed may impose such constraints.

