工厂方法（FactoryMethod）模式的定义：定义一个创建产品对象的工厂接口，将产品对象的实际创建工作推迟到具体子工厂类当中。这满足创建型模式中所要求的“创建与使用相分离”的特点。

我们把被创建的对象称为“产品”，把创建产品的对象称为“工厂”。如果要创建的产品不多，只要一个工厂类就可以完成，这种模式叫“简单工厂模式”，它不属于 GoF 的 23 种经典设计模式，它的缺点是增加新产品时会违背“开闭原则”。

本节介绍的“工厂方法模式”是对简单工厂模式的进一步抽象化，其好处是可以使系统在不修改原来代码的情况下引进新的产品，即满足开闭原则。

工厂方法模式的主要优点有：
- **用户只需要知道具体工厂的名称就可得到所要的产品，无须知道产品的具体创建过程；**
- **在系统增加新的产品时只需要添加具体产品类和对应的具体工厂类，无须对原工厂进行任何修改，满足开闭原则；**

其缺点是：每增加一个产品就要增加一个具体产品类和一个对应的具体工厂类，这增加了系统的复杂度。


工厂方法模式的主要角色：

1. 模式的结构抽象工厂（Abstract Factory）：提供了创建产品的接口，调用者通过它访问具体工厂的工厂方法 newProduct() 来创建产品。
2. 具体工厂（ConcreteFactory）：主要是实现抽象工厂中的抽象方法，完成具体产品的创建。
3. 抽象产品（Product）：定义了产品的规范，描述了产品的主要特性和功能。
4. 具体产品（ConcreteProduct）：实现了抽象产品角色所定义的接口，由具体工厂来创建，它同具体工厂之间一一对应。

```java
// 抽象产品
interface Product {
    void show();
}

// 具体产品
class Book implements Product {
    @Override
    public void show() {
        System.out.println("this is a Book!");
    }
}

// 抽象工厂
interface AbstractFactory {
  	// 有的时候 一个Factory 中也可以定义多个工厂方法，这类工厂称为"抽象工厂"
    Product newProduct();
}

// 具体工厂
class BookFactory implements AbstractFactory {
    public Product newProduct() {
        System.out.println("BookFactory create book...");
        return new Book();
    }
}

class Factory {
    public static void main(String[] args) {
        AbstractFactory factory = new BookFactory();
        Product p = factory.newProduct();
        p.show();
    }
}
```

> 输出结果：
>
> BookFactory create book...
> this is a Book!