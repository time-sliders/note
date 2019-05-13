## 基于ClassLoader 实现 Hot-Deployment存在的问题

在 Java 中，即使是同一个类文件，如果是由不同的类加载器实例加载的，那么它们的类型是不相同的。如果认为是同一个类，就出现ClassCastException异常，也就类转换异常。这就意味着，我们在实现热替换的时候，我们自定义的classLoader所生产的类对象，不能被赋值到原项目里的所对应的类对象。

> 为什么不直接让系统 classLoader 重新加载 Class 文件？
> ClassLoader 对象是不允许相同 classs 文件重新加载

## 解决方案

#### 一 如何解决 ClassLoader 不能重复加载类的问题？

**使用两个 ClassLoader，一个 Classloader 加载那些不会改变的类（第三方Jar包），另一个ClassLoader加载会更改的类，称为 restart ClassLoader ，在有代码更改的时候，原来的restart ClassLoader 被丢弃，重新创建一个restart ClassLoader**

#### 二 如何解决 ClassLoader 加载类之后出现 ClassCastException 的问题？

用 AppClassLoader (or SystemClassLoader 加载 Interface)，用自定义 ClassLoader 加载 Intf-Impl 实现类。在使用的时候，只通过接口的方式去使用。



#### 现网问题

不建议再生产环境上使用热部署，因为性能问题和很多潜在问题都无法预料。

1. ClassLoader GC如何回收问题？

