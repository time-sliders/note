单例（Singleton）模式的定义：指一个类只有一个实例，且该类能自行创建这个实例的一种模式。

```java
public class Singleton {

    // 单例实例
    private static volatile Singleton instance;

    // 构造器私有化
    private Singleton() {
    }

    public static Singleton getInstance() {
        if (instance == null) {
          	// 当第一次调用 getlnstance 方法时才去创建这个单例
            synchronized (Singleton.class) { // 并发锁
                // 双重检查
                if (instance == null) {
                    // 初始化单例实例
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

```java
public class Singleton {

    // 类一旦加载就创建一个单例实例 保证在调用 getInstance 方法之前单例已经存在
    private static final Singleton instance = new Singleton();

    // 构造器私有化
    private Singleton() {
    }

    public static Singleton getInstance() {
        return instance;
    }
}
```