A token representing the registration of a `SelectableChannel` with a `Selector`.
一个用于描述 SelectableChannel 在 Selector 上注册信息的 token。

A selection key is created each time a channel is registered with a selector. A key remains valid until it is *cancelled* by invoking its `cancel` method, by closing its channel, or by closing its selector. Cancelling a key does not immediately remove it from its selector; it is instead added to the selector's `cancelled-key set` for removal during the next selection operation. The validity of a key may be tested by invoking its `isValid` method. 
SelectionKey 是在 channel 每次注册到 Selector 上创建的。一个 SelectionKey 将一直保持有效，直到 channel 或者 selector 关闭导致它的 cancel 方法被调用。 cancel 一个 SelectionKey 并不会立即将它从 Selector 上移除掉，而是添加到 Selector 的 cancelled-key 集合上去，然后在下一次 selection 操作进行时，将会被移除。可以通过 isValid 方法来检测一个 key 的有效性。

A selection key contains two operation sets represented as integer values. Each bit of an operation set denotes a category of selectable operations that are supported by the key's channel.
一个 SelectionKey 通过一个 integer 值来呈现出 2 种状态集合，操作集合上的每一个bit 表示这个 key 的channel 所支持的一类可以选择的操作。


* The **interest set** determines which operation categories will be tested for readiness the next time one of the selector's selection methods is invoked. The interest set is initialized with the value given when the key is created; it may later be changed via the `interestOps(int)` method.
  **interest set **决定在下一次 selector 的 selection 方法被调用时，哪些操作类型将会被检测是否就绪。interest-set 是根据创建时传入的 value 来初始化的；在后续可以通过“interestop（int）”方法更改它。
* The **ready set** identifies the operation categories for which the key's channel has been detected to be ready by the key's selector. The ready set is initialized to zero when the key is created; it may later be updated by the selector during a selection operation, but it cannot be updated directly.
  ready-set 

That a selection key's ready set indicates that its channel is ready for some operation category is a hint, but not a guarantee, that an operation in such a category may be performed by a thread without causing the thread to block. A ready set is most likely to be accurate immediately after the completion of a selection operation. It is likely to be made inaccurate by external events and by I/O operations that are invoked upon the corresponding channel.

This class defines all known operation-set bits, but precisely which bits are supported by a given channel depends upon the type of the channel. Each subclass of `SelectableChannel` defines an `validOps()` method which returns a set identifying just those operations that are supported by the channel. An attempt to set or test an operation-set bit that is not supported by a key's channel will result in an appropriate run-time exception.

It is often necessary to associate some application-specific data with a selection key, for example an object that represents the state of a higher-level protocol and handles readiness notifications in order to implement that protocol. Selection keys therefore support the attachment of a single arbitrary object to a key. An object can be attached via the `attach` method and then later retrieved via the `attachment` method.

Selection keys are safe for use by multiple concurrent threads. The operations of reading and writing the interest set will, in general, be synchronized with certain operations of the selector. Exactly how this synchronization is performed is implementation-dependent: In a naive implementation, reading or writing the interest set may block indefinitely if a selection operation is already in progress; in a high-performance implementation, reading or writing the interest set may block briefly, if at all. In any case, a selection operation will always use the interest-set value that was current at the moment that the operation began.