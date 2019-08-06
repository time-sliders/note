A multiplexor of `SelectableChannel` objects.
一个 SelectableChannel 对象的多路复用器

A selector may be created by invoking the `open` method of this class, which will use the system's default `selector provider` to create a new selector. A selector may also be created by invoking the openSelector method of a custom selector provider. A selector remains open until it is closed via its `close` method. 
可以通过该类的 open 方法创建一个 selector，将会使用系统默认的 selector provider 来创建一个新的 selector。也可以通过自定义的 selector provider 来创建一个 selector，一个selector 将会一直保持有效，直到它的 close 方法被调用。

A selectable channel's registration with a selector is represented by a `SelectionKey` object. A selector maintains three sets of selection keys:
一个 selectable channel 在 selector 上的注册信息是通过 SelectionKey 对象来表现的，一个 selector 维护三个 selecion keys 的集合

* The ***key set*** contains the keys representing the current channel registrations of this selector. This set is returned by the keys method.
  key set 包含那些用于表示当前 channel 在 selector 上的注册信息的 keys，该集合可以通过 keys 方法返回
* The ***selected-key set*** is the set of keys such that each key's channel was detected to be ready for at least one of the operations identified in the key's interest set during a prior selection operation. This set is returned by the `selectedKeys` method. The selected-key set is always a subset of the key set.
  ***selected-key set*** 是那些每个 key's 对应的 channel 已经在当前 key 的 interest-set 中指定的至少一个操作在之前的一个 selection 操作时被发现已经准备检测就绪的 key 的集合。这个集合通过 selectedKeys 方法返回，selected-Key 集合总是 key-set 的一个子集。
* The ***cancelled-key set*** is the set of keys that have been cancelled but whose channels have not yet been deregistered. This set is not directly accessible. The cancelled-key set is always a subset of the key set.
  ***cancelled-key set*** 是那些已经被 cancelled 但是对应的 channel 还没有注销掉的 keys 集合。

All three sets are empty in a newly-created selector.
在一个新创建的 selector 里，这三个集合都是空的。

A key is added to a selector's key set as a side effect of registering a channel via the channel's `register` method. Cancelled keys are removed from the key set during selection operations. The key set itself is not directly modifiable.
channel 的 register 方法注册一个 channel 到 selector 上会导致 一个key 添加到 selector 的 key-set 。Cancelled keys 会在 selection 操作时从 key-set 上移除掉，key-set 本身不是直接可修改的。

A key is added to its selector's cancelled-key set when it is cancelled, whether by closing its channel or by invoking its cancel method. Cancelling a key will cause its channel to be deregistered during the **next** selection operation, at which time the key will removed from all of the selector's key sets. 
一个key 在cancelled 的时候会添加到 selector 的 cancelled-key 集合上，无论是通过关闭 channel 或者它的 close 方法。cancelling 一个 key 会导致它的 channel 在下一次 selection 操作时被注销，同时key 将会从 selector 的 所有 key 集合上移除掉。

Keys are added to the selected-key set by selection operations. A key may be removed directly from the selected-key set by invoking the set's remove method or by invoking the remove method of an iterator obtained from the set. Keys are never removed from the selected-key set in any other way; they are not, in particular, removed as a side effect of selection operations. Keys may not be added directly to the selected-key set.
keys 通过 selection 操作添加到 selected-key 集合上去，一个 key 可能会通过调用 set 的 remove 方法或者调用 set Iterator 的remove 方法，从而被直接从 seleced-key 集合上移除。keys 永远不会通过其他方式从 selected-key 集合上移除，尤其是，它们不会在 selection 操作的时候被删除。键不能直接添加到选定的键集。keys 不能直接的添加到 selected-key 集合上去。

# Selection
During each selection operation, keys may be added to and removed from a selector's selected-key set and may be removed from its key and cancelled-key sets. Selection is performed by the `select()`, `select(long),` and `selectNow()` methods, and involves three steps:
在每一次 selection 操作的时候，key 可能会从一个 selector 的 selected-key 集合上添加/移除，也有可能从它的 key-set 或者 cancelled-key set 上移除。Selection 是通过`select()`, `select(long),` and `selectNow()`  三个方法执行的，调用包含如下三步：

1. Each key in the cancelled-key set is removed from each key set of which it is a member, and its channel is deregistered. This step leaves the cancelled-key set empty.
   cancelled-key 集合上的每一个 key 将会从所有 key-set 上移除，相应的 channel 也会注销，这一步在 cancelled-key 集合为空时退出。

2. The underlying operating system is queried for an update as to the readiness of each remaining channel to perform any of the operations identified by its key's interest set as of the moment that the selection operation began. For a channel that is ready for at least one such operation, one of the following two actions is performed:
   在选择操作开始时，将询问底层操作系统，在 selection 操作开始的时候，剩余的 channel 是否已经在它们感兴趣的操作集合上准备就绪。对于一个已经在至少一个这样的操作上准备就绪的通道，将执行以下两个操作之一：

   1. If the channel's key is not already in the selected-key set then it is added to that set and its ready-operation set is modified to identify exactly those operations for which the channel is now reported to be ready. Any readiness information previously recorded in the ready set is discarded.
      如果通道的 key 还未在 selected-key 集合中，则会将其添加到该集合中，并修改其 ready-operation set ，以准确识别通道现在报告就绪的操作。先前记录在 ready set 中的任何就绪信息都将被丢弃。
   2. Otherwise the channel's key is already in the selected-key set, so its ready-operation set is modified to identify any new operations for which the channel is reported to be ready. Any readiness information previously recorded in the ready set is preserved; in other words, the ready set returned by the underlying system is bitwise-disjoined into the key's current ready set.
      如果 channel 的 key 已经在 selected-key 集合上，所以它的  ready-operation set 会被修改为标识当前通道已经报告就绪的新的操作集合，在这之前纪录的任何就绪信息会被保留；换句话说，底层操作系统返回的 ready-set 已经按位分开合并到key的当前就绪集合上。

   If all of the keys in the key set at the start of this step have empty interest sets then neither the selected-key set nor any of the keys' ready-operation sets will be updated.
   如果所有key set 中所有的 key 都没有 interest 集合，那么selected-set 与 ready-set 都不会被更新

3. If any keys were added to the cancelled-key set while step (2) was in progress then they are processed as in step (1).
   如果任何key 在第二步处理的时候被添加到 cancelled-key 集合上去，那么它们将会被下一个 selection 的第一步操作处理掉。

Whether or not a selection operation blocks to wait for one or more channels to become ready, and if so for how long, is the only essential difference between the three selection methods.
选择操作是否阻止等待一个或多个通道准备就绪，如果是，等待多长时间，这是三种选择方法之间唯一的本质区别。

# Concurrency
Selectors are themselves safe for use by multiple concurrent threads; their key sets, however, are not.
Selector 本身是线程安全的，但是它的 key set 并不是

The selection operations synchronize on the selector itself, on the key set, and on the selected-key set, in that order. They also synchronize on the cancelled-key set during steps (1) and (3) above.
选择操作在 selector 本身、key-set和selected-key set上按该顺序同步。在上述步骤（1）和（3）中，它们还在 cancelled-key set 上同步。

Changes made to the interest sets of a selector's keys while a selection operation is in progress have no effect upon that operation; they will be seen by the next selection operation.
在进行选择操作时对选择器键的兴趣集所做的更改对该操作没有影响；下一个选择操作将看到这些更改。

Keys may be cancelled and channels may be closed at any time. Hence因此 the presence(存在) of a key in one or more of a selector's key sets does not imply that the key is valid or that its channel is open. Application code should be careful to synchronize and check these conditions as necessary if there is any possibility that another thread will cancel a key or close a channel.
key 可以随时取消，channel 也可以随时和关闭。因此，在选择器的一个或多个key-set中存在并不意味着该key是有效的或其channel是打开的。如果另一个线程可能会取消某个键或关闭某个通道，那么应用程序代码应该小心地同步和检查这些条件。

A thread blocked in one of the `select()` or `select(long)` methods may be interrupted by some other thread in one of three ways:
在“select”或“select(long)”方法之一中阻塞的线程可能会被其他线程以三种方式之一中断：

* By invoking the selector's `wakeup` method,
* By invoking the selector's `close` method, or
* By invoking the blocked thread's `interrupt` method, in which case its interrupt status will be set and the selector's `wakeup` method will be invoked.

The `close` method synchronizes on the selector and all three key sets in the same order as in a selection operation. 
“close”方法在选择器和所有三个键集上同步的顺序与选择操作中的顺序相同。

A selector's key and selected-key sets are not, in general, safe for use by multiple concurrent threads. If such a thread might modify one of these sets directly then access should be controlled by synchronizing on the set itself. The iterators returned by these sets' iterator methods are fail-fast: If the set is modified after the iterator is created, in any way except by invoking the iterator's own `remove` method, then a `java.util.ConcurrentModificationException` will be thrown.
通常，选择器的键和选定的键集对于多个并发线程的使用不安全。如果这样的线程可以直接修改其中一个集合，那么应该通过在该集上同步来控制访问。这些迭代器方法返回的迭代器是快速失败的：如果在迭代器创建之后修改了集合，除了调用迭代器自己的“Read”方法外，将抛出一个“java.util.ConcurrentModificationException”。