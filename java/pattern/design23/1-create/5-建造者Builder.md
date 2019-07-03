在软件开发过程中有时需要创建一个复杂的对象，这个复杂对象通常由多个子部件按一定的步骤组合而成。例如，计算机是由 OPU、主板、内存、硬盘、显卡、机箱、显示器、键盘、鼠标等部件组装而成的，采购员不可能自己去组装计算机，而是将计算机的配置要求告诉计算机销售公司，计算机销售公司安排技术人员去组装计算机，然后再交给要买计算机的采购员。

**以上所有这些产品都是由多个部件构成的，各个部件可以灵活选择，但其创建步骤都大同小异**。这类产品的创建无法用前面介绍的工厂模式描述，只有建造者模式可以很好地描述该类产品的创建。

## 模式的定义与特点

建造者（Builder）模式的定义：指将一个复杂对象的构造与它的表示分离，使同样的构建过程可以创建不同的表示，这样的被称为建造者模式。它是将一个复杂的对象分解为多个简单的对象，然后一步一步构建而成。它将变与不变相分离，即产品的组成部分是不变的，但每一部分是可以灵活选择的。

该模式的主要优点如下：

1. 各个具体的建造者相互独立，有**利于系统的扩展**。
2. 客户端不必知道产品内部组成的细节，便于控制细节风险。

其缺点如下：
1. 产品的组成部分必须相同，这限制了其使用范围。
2. 如果产品的内部变化复杂，该模式会增加很多的建造者类。

建造者（Builder）模式和工厂模式的关注点不同：**建造者模式注重零部件的组装过程**，而工厂方法模式更注重零部件的创建过程，但两者可以结合使用。

## 模式的结构与实现

建造者（Builder）模式由产品、抽象建造者、具体建造者、指挥者等 4 个要素构成，现在我们来分析其基本结构和实现方法。

#### 1. 模式的结构

建造者（Builder）模式的主要角色如下。

1. 产品角色（Product）：它是包含多个组成部件的复杂对象，由具体建造者来创建其各个滅部件。
2. 抽象建造者（Builder）：它是一个包含创建产品各个子部件的抽象方法的接口，通常还包含一个返回复杂产品的方法 getResult()。
3. 具体建造者(Concrete Builder）：实现 Builder 接口，完成复杂产品的各个部件的具体创建方法。
4. 指挥者（Director）：它调用建造者对象中的部件构造与装配方法完成复杂对象的创建，在指挥者中不涉及具体产品的信息。

```java
class Car {
    int engine;
    int wheel;
    int body;

    @Override
    public String toString() {
        return "Car{" +
                "engine=" + engine +
                ", wheel=" + wheel +
                ", body=" + body +
                '}';
    }
}


public class Builder {

    private Car car;

    public Builder() {
        car = new Car();
    }

    public Builder setEngine(int i) {
        car.engine = i;
        return this;
    }

    public Builder setWheel(int i) {
        car.wheel = i;
        return this;
    }

    public Builder setBody(int i) {
        car.body = i;
        return this;
    }

    public Car build() {
        return car;
    }

    public static void main(String[] args) {
        Car c = new Builder().setBody(1).setEngine(2).setWheel(4).build();
        System.out.println(c.toString());
    }
}
```