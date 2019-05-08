原文地址: [concurrent-mark-sweep-cms-collector](https://docs.oracle.com/javase/10/gctuning/concurrent-mark-sweep-cms-collector.htm)

## Concurrent Mark Sweep Collector Performance and Structure

Similar to the other available collectors, the CMS collector is **generational**; thus both **minor** and **major** collections occur. The CMS collector attempts to **reduce pause times due to major collections** by **using separate garbage collector threads to <u>trace the reachable objects concurrently</u> with the execution of the application threads**.

During each major collection cycle, the CMS collector pauses all the application threads for a brief period at the **beginning** of the collection and again toward the **middle** of the collection. The second pause tends to be`往往是` the longer of the two pauses. Multiple threads perform the collection work during both pauses. One or more garbage collector threads do the remainder of the collection (including most of the tracing of live objects and sweeping of unreachable objects). Minor collections can interleave with`交织` an ongoing major cycle, and are done in a manner`方式` similar to the parallel collector (in particular, the application threads are stopped during minor collections).



## Concurrent Mode Failure

The CMS collector uses **one or more garbage collector threads** that run simultaneously`同时地` with the application threads with the goal of`目的是` **completing the collection of the old generation before it becomes full**.

As described previously, in normal operation, the CMS collector does most of its **tracing** and **sweeping** work with the application threads still running, so only brief pauses are seen by the application threads. However, **if the CMS collector is unable to finish reclaiming the unreachable objects before the old generation fills up, or if an allocation cannot be satisfied with the available free space blocks in the old generation, then the application is paused and the collection is completed with all the application threads stopped**`如果CMS收集器无法在老年代填满之前完成对不可访问对象的回收，或者如果老年代代中的可用空间块无法满足分配要求，则应用程序将暂停，并在所有应用程序线程停止的情况下完成收集`. The inability `没有能力` to complete a collection concurrently is referred to as`被称为` *concurrent mode failure* and indicates`表明` the need to adjust the CMS collector parameters. If a concurrent collection is interrupted by an explicit`显示的` garbage collection (System.gc()) or for a garbage collection needed to provide information for diagnostic tools, then a concurrent mode interruption is reported.



## Excessive`过长的` GC Time and `OutOfMemoryError`

The CMS collector throws an `OutOfMemoryError` if too much time is being spent in garbage collection: If more than 98% of the total time is spent in garbage collection and less than 2% of the heap is recovered, then an `OutOfMemoryError` is thrown.

This feature is designed to prevent applications from running for an extended period of time while making little or no progress because the heap is too small. If necessary, this feature can be disabled by adding the option `-XX:-UseGCOverheadLimit` to the command line.

The policy is the same as that in the parallel collector, except that time spent performing concurrent collections isn't counted toward the 98% time limit. In other words, only collections performed while the application is stopped count toward excessive GC time. Such collections are typically due to a concurrent mode failure or an explicit collection request (for example, a call to `System.gc()`).



## Concurrent Mark Sweep Collector and Floating Garbage

The CMS collector, like all the other collectors in Java HotSpot VM, is a **tracing collector** that **identifies`标识` at least all the reachable`可访问` objects in the heap.**

Richard Jones and Rafael D. Lins in their publication *Garbage Collection: Algorithms for Automated Dynamic Memory*, it's an **incremental update collector**. Because application threads and the garbage collector thread run concurrently during a major collection, objects that are traced by the garbage collector thread may subsequently become unreachable by the time collection process ends. Such unreachable objects that haven't yet been reclaimed are referred to as *floating garbage*`漂浮垃圾`. The amount of floating garbage depends on the duration of the concurrent collection cycle and on the frequency of reference updates, also known as *mutations*, by the application. Furthermore`此外`, because the young generation and the old generation are collected independently, each acts as a source of roots to the other`每一个都是彼此的根源`. As a rough guideline`作为一个粗略的指导方针`, try increasing the size of the old generation by 20% to account for`比例上占` the floating garbage`根据经验建议将老年代增加20%的空间来承载浮动垃圾`. **Floating garbage in the heap at the end of one concurrent collection cycle is collected during the next collection cycle.**



## Concurrent Mark Sweep Collector Pauses

The CMS collector pauses an application twice during a concurrent collection cycle. **The first pause is to mark as live the objects directly reachable from the <u>roots</u>** (for example, object references from **application thread stack**s and **registers`寄存器`**, **static objects**, and so on) and from elsewhere in the heap (for example, the **young generation**).

This first pause is referred to as the ***initial mark pause***. The second pause comes at the end of the concurrent tracing phase and finds objects that were missed by the concurrent tracing due to updates by the application threads of references in an object after the CMS collector had finished tracing that object. This second pause is referred to as the ***remark pause***.



## Concurrent Mark Sweep Collector Concurrent Phases

The **concurrent tracing of the reachable object graph** occurs between the **initial mark pause** and the **remark pause**.

<u>During this concurrent tracing phase, one or more concurrent garbage collector threads may be using processor resources that would otherwise have been available to the application. As a result, compute-bound`受计算量限制的` applications may see a commensurate`相当,相称的` decrease in application throughput during this and other concurrent phases even though the application threads aren’t paused. After the remark pause, a concurrent sweeping phase collects the objects identified as unreachable. After a collection cycle completes, the CMS collector waits, consuming almost no computational resources`几乎不消耗计算资源`, until the start of the next major collection cycle.</u>



## Starting a Concurrent Collection Cycle

With the *serial collector* a major collection occurs whenever the old generation becomes full and all application threads are stopped while the collection is done. In contrast与之相反, <u>the start of a concurrent collection in CMS collector must be **timed** such that the collection can finish before the old generation becomes full; otherwise, the application would observe **longer pauses due to concurrent mode failure**. There are several ways to start a concurrent collection.</u> 

Based on recent`最近的` history, the CMS collector maintains **estimates**`估计` of the time remaining before the old generation will be exhausted and of the time needed for a concurrent collection cycle. Using these dynamic estimates, a concurrent collection cycle is started with the aim of`目的` completing the collection cycle before the old generation is exhausted. These estimates are padded for safety because concurrent mode failure can be very costly`为了安全起见，这些估值被padded，因为 concurrent mode failure 的代价非常昂贵.`

基于最近的回收历史，CMS会维护两个预估时间：老年代多久会被耗尽；完成一次老年代的回收需要多久。依据这些动态估计值，在老年代耗尽前适时开始老年代的回收。这些预估值是为了避免耗时的**concurrent mode failure**。

A concurrent collection also starts if the occupancy of the old generation exceeds an initiating occupancy (a percentage of the old generation). The default value for this initiating occupancy threshold is approximately 92%, but the value is subject to change from release to release. This value can be manually adjusted using the command-line option `-XX:CMSInitiatingOccupancyFraction=``<N>`, where `<N>` is an integral percentage (0 to 100) of the old generation size.



## Scheduling Pauses

The pauses for the young generation collection and the old generation collection occur independently.

They don't overlap`重叠`, but may occur in quick succession`快速连续` such that the pause from one collection, immediately followed by one from the other collection, can appear to be a single, longer pause. To avoid this, the CMS collector attempts to schedule the remark pause roughly midway between`大体在…中间` the previous and next young generation pauses.`尝试规划将remark暂停放在两次新生代回收（Young GC）之间` This scheduling is currently not done for the initial mark pause, which is usually much shorter than the remark pause.



## Concurrent Mark Sweep Collector Measurements

The following is the output from the CMS collector with the option **`-Xlog:gc`**:


>[121,834s][info][gc] GC(657) **Pause Initial Mark** 191M->191M(485M) (121,831s, 121,834s) 3,433ms
[121,835s][info][gc] GC(657) Concurrent Mark (121,835s)
[121,889s][info][gc] GC(657) Concurrent Mark (121,835s, 121,889s) 54,330ms
[121,889s][info][gc] GC(657) Concurrent Preclean (121,889s)
[121,892s][info][gc] GC(657) Concurrent Preclean (121,889s, 121,892s) 2,781ms
[121,892s][info][gc] GC(657) Concurrent Abortable Preclean (121,892s)
[121,949s][info][gc] GC(658) Pause Young (Allocation Failure) 324M->199M(485M) (121,929s, 121,949s) 19,705ms
[122,068s][info][gc] GC(659) Pause Young (Allocation Failure) 333M->200M(485M) (122,043s, 122,068s) 24,892ms
[122,075s][info][gc] GC(657) Concurrent Abortable Preclean (121,892s, 122,075s) 182,989ms
[122,087s][info][gc] GC(657) **Pause Remark** 209M->209M(485M) (122,076s, 122,087s) 11,373ms
[122,087s][info][gc] GC(657) Concurrent Sweep (122,087s)
[122,193s][info][gc] GC(660) Pause Young (Allocation Failure) 301M->165M(485M) (122,181s, 122,193s) 12,151ms
[122,254s][info][gc] GC(657) Concurrent Sweep (122,087s, 122,254s) 166,758ms
[122,254s][info][gc] GC(657) Concurrent Reset (122,254s)
[122,255s][info][gc] GC(657) Concurrent Reset (122,254s, 122,255s) 0,952ms
[122,297s][info][gc] GC(661) Pause Young (Allocation Failure) 259M->128M(485M) (122,291s, 122,297s) 5,797ms


Note:

The output for the CMS collection (GC ID 657) is interspersed`散布` with the output from the minor collections (GC IDs 658, 659 and 660); typically many minor collections occur during a concurrent collection cycle. `Pause Initial Mark` indicates the start of the concurrent collection cycle. The lines starting with "`Concurrent`" indicate the start and end of the concurrent phases. `Pause Remark` is the final pause. Not discussed previously is the precleaning phases. `Precleaning` represents work that can be done concurrently in preparation for the remark phase. The final phase is indicated by `Concurrent Reset` and is in preparation for the next concurrent collection.

The `initial mark pause` is typically short relative to the minor collection pause time. The concurrent phases (concurrent mark, concurrent preclean, and concurrent sweep) normally last significantly longer than a minor collection pause, as indicated in the CMS collector output example. Note, however, **that the application isn't paused during these concurrent phases**`不难发现，并发阶段（并发标记，并发预清理和并发清除）通常耗时比新生代回收时间长。但是，应用在这些阶段中并没有暂停`. The **remark pause** is often comparable in length to a minor collection. The remark pause is affected by certain application characteristics (for example, a high rate of object modification can increase this pause) and the time since the last minor collection (for example, more objects in the young generation may increase this pause).`重新标记的耗时会受某些应用特性（比如：高对象修改率会增加此阶段耗时）和距离上次老年代收集时间（比如：新生代有了更多的对象会增加此阶段耗时）的影响。`