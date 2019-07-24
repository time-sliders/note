用一个已经创建的实例作为原型，通过复制该原型对象来创建一个和原型相同或相似的新对象。由于 [Java](http://c.biancheng.net/java/) 提供了对象的 clone() 方法，所以用 Java 实现原型模式很简单。

原型模式的拷贝分为 **浅拷贝** 和 **深拷贝**，Java 中的 Object 类提供了浅拷贝的 clone() 方法，具体原型类只要实现 Cloneable 接口就可实现对象的浅拷贝，这里的 Cloneable 接口就是抽象原型类。其代码如下：

```java
public class Prototype implements Cloneable {

    private String name = "zhw";

    public static void main(String[] args) throws CloneNotSupportedException {
        Prototype p = new Prototype();
        Prototype c = (Prototype) p.clone();
        System.out.println(c.name + " " + (c == p));
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

> 输出结果： zhw false

# Cloneable JavaDoc

A class implements the Cloneable interface to indicate to the Object.clone() method that it is legal for that method to **make a <u>field-for-field copy</u> of instances of that class**.
<u>Invoking Object's clone method on an instance that does not implement the Cloneable interface results in the exception CloneNotSupportedException being thrown.</u>
By convention按惯例, **classes that implement this interface should override Object.clone (which is protected) with a public method. See Object.clone() for details on overriding this method.**
Note that this interface does not contain the clone method. Therefore, it is not possible to clone an object merely仅仅 by virtue of凭借 the fact that it implements this interface. Even if the clone method is invoked reflectively反射地, there is no guarantee that it will succeed
