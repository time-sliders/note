**primitive type**:   基本类型，像int、double就是。
**wrapped type**:	包装类型，int—>Integer，double—>Decimal

基本类型不可实例化，可以直接初始化、赋值、运算。
包装类型就是把基本类型变成一个类实例，一定要new才产生。

堆栈里分别存放什么东西：
栈存储运行时声明的变量 —— 对象引用（或基础类型, primitive）内存空间。
堆分配每一个对象内容（实例）内存空间。

栈的实现是先入后出的。
堆是随机存放的。

StackOverFlow : 总是在无限递归调用时候可以看见。
OutOfMemory   : 可以通过无限 new 实现。