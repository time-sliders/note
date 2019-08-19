Vector 继承自 ArrayList , 实现了线程安全（重写父类的所有方法，并通过 Synchronzied 关键字修饰）。

另外添加个一个 capacityIncrement 变量，在扩容时，可以根据该字段进行扩容，用于减少扩容次数。

```java
int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                 capacityIncrement : oldCapacity);
```

（ArrayList 是直接进行 “增加 50% 容量” 来进行扩容的：`int newCapacity = oldCapacity + (oldCapacity >> 1);`）

