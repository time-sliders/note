cglib是一个java 字节码的生成工具，它是对asm的进一步封装。研究cglib主要是因为它也提供了动态代理功能，这点和jdk的动态代理类似。

# cglib 动态代理示例  

1.  **创建目标类**

   ```java
   public class Target {
       public void f() {
           System.out.println("Target.f() invoked");
       }
   }
   ```

2. **创建拦截器**

   ```java
   public class Interceptor implements MethodInterceptor {
       @Override
       public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
           System.out.println("I am intercept begin");
           //Note: 此处一定要使用 proxy 的 invokeSuper 方法来调用目标类的方法
           proxy.invokeSuper(obj, args);
           System.out.println("I am intercept end");
           return null;
       }
   }
   ```

3. **测试**

   ```java
   public class TestCglib {
       public static void main(String[] args) {
           System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY,"/Users/zhangwei/Downloads/");
           //实例化一个增强器，也就是cglib中的一个class generator
           Enhancer enhancer = new Enhancer();
           //设置目标类
           enhancer.setSuperclass(Target.class);
           // 设置拦截对象
           enhancer.setCallback(new Interceptor());
           // 生成代理类并返回一个实例
           Target t = (Target) enhancer.create();
           t.f();
       }
   }
   ```

4.  输出结果

   > I am intercept begin
   > Target.f() invoked
   > I am intercept end

与JDK动态代理相比，**cglib可以实现对一般类的代理而无需实现接口**。在上例中通过下列步骤来生成目标类Target的代理类：

1. 创建`Enhancer`实例
2. 通过`setSuperclass`方法来设置目标类
3. 通过`setCallback` 方法来设置拦截对象
4. `create`方法生成`Target`的代理类，并返回代理类的实例

# 二、代理类分析

在示例代码中我们通过设置`DebuggingClassWriter.DEBUG_LOCATION_PROPERTY`的属性值来获取cglib生成的代理类。通过之前分析的命名规则我们可以很容易的在 */Users/zhangwei/Downloads/* 下面找到生成的代理类`Target$$EnhancerByCGLIB$$cdd37a52.class`

使用Idea进行反编译，由于cglib会代理Object中的`finalize`,`equals`, `toString`,`hashCode`,`clone`方法，为了清晰的展示代理类我们省略这部分代码，反编译的结果如下

```java
package com.noob.storage.aop.cglib;

import java.lang.reflect.Method;
import net.sf.cglib.core.ReflectUtils;
import net.sf.cglib.core.Signature;
import net.sf.cglib.proxy.Callback;
import net.sf.cglib.proxy.Factory;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

public class Target$$EnhancerByCGLIB$$cdd37a52 extends Target implements Factory {
    private boolean CGLIB$BOUND;
    public static Object CGLIB$FACTORY_DATA;
    private static final ThreadLocal CGLIB$THREAD_CALLBACKS;
    private static final Callback[] CGLIB$STATIC_CALLBACKS;
    private MethodInterceptor CGLIB$CALLBACK_0;
    private static Object CGLIB$CALLBACK_FILTER;
    private static final Method CGLIB$f$0$Method;
    private static final MethodProxy CGLIB$f$0$Proxy;
    private static final Object[] CGLIB$emptyArgs;
    private static final Method CGLIB$equals$1$Method;
    private static final MethodProxy CGLIB$equals$1$Proxy;
    private static final Method CGLIB$toString$2$Method;
    private static final MethodProxy CGLIB$toString$2$Proxy;
    private static final Method CGLIB$hashCode$3$Method;
    private static final MethodProxy CGLIB$hashCode$3$Proxy;
    private static final Method CGLIB$clone$4$Method;
    private static final MethodProxy CGLIB$clone$4$Proxy;

    static void CGLIB$STATICHOOK1() {
        CGLIB$THREAD_CALLBACKS = new ThreadLocal();
        CGLIB$emptyArgs = new Object[0];
        Class var0 = Class.forName("com.noob.storage.aop.cglib.Target$$EnhancerByCGLIB$$cdd37a52");
        Class var1;
        Method[] var10000 = ReflectUtils.findMethods(new String[]{"equals", "(Ljava/lang/Object;)Z", "toString", "()Ljava/lang/String;", "hashCode", "()I", "clone", "()Ljava/lang/Object;"}, (var1 = Class.forName("java.lang.Object")).getDeclaredMethods());
        CGLIB$equals$1$Method = var10000[0];
        CGLIB$equals$1$Proxy = MethodProxy.create(var1, var0, "(Ljava/lang/Object;)Z", "equals", "CGLIB$equals$1");
        CGLIB$toString$2$Method = var10000[1];
        CGLIB$toString$2$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/String;", "toString", "CGLIB$toString$2");
        CGLIB$hashCode$3$Method = var10000[2];
        CGLIB$hashCode$3$Proxy = MethodProxy.create(var1, var0, "()I", "hashCode", "CGLIB$hashCode$3");
        CGLIB$clone$4$Method = var10000[3];
        CGLIB$clone$4$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/Object;", "clone", "CGLIB$clone$4");
        CGLIB$f$0$Method = ReflectUtils.findMethods(new String[]{"f", "()V"}, (var1 = Class.forName("com.noob.storage.aop.cglib.Target")).getDeclaredMethods())[0];
        CGLIB$f$0$Proxy = MethodProxy.create(var1, var0, "()V", "f", "CGLIB$f$0");
    }

    final void CGLIB$f$0() {
        super.f();
    }

    public final void f() {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            var10000.intercept(this, CGLIB$f$0$Method, CGLIB$emptyArgs, CGLIB$f$0$Proxy);
        } else {
            super.f();
        }
    }
}
```

 代理类（`Target$$EnhancerByCGLIB$$cdd37a52`）**继承**了目标类（`Target`），至于代理类实现的`factory`接口与本文无关。**代理类为每个目标类的方法生成两个方法，例如针对目标类中的每个非`private`方法，代理类会生成两个方法，以`f`方法为例：一个是`@Override`的`f`方法，一个是`CGLIB$f$0`（`CGLIB$f$0`相当于目标类的`f`方法）。我们在示例代码中调用目标类的方法`t.f()`时，实际上调用的是代理类中的`f()`方法**。接下来我们着重分析代理类中的`f`方法，看看是怎么实现的代理功能。

​      当调用代理类的`f`方法时，先判断是否已经存在实现了`MethodInterceptor`接口的拦截对象，如果没有的话就调用`CGLIB$BIND_CALLBACKS`方法来获取拦截对象，`CGLIB$BIND_CALLBACKS`的反编译结果如下：

```java
private static final void CGLIB$BIND_CALLBACKS(Object var0) {
  Target$$EnhancerByCGLIB$$cdd37a52 var1 = (Target$$EnhancerByCGLIB$$cdd37a52)var0;
  if (!var1.CGLIB$BOUND) {
    var1.CGLIB$BOUND = true;
    Object var10000 = CGLIB$THREAD_CALLBACKS.get();
    if (var10000 == null) {
      var10000 = CGLIB$STATIC_CALLBACKS;
      if (var10000 == null) {
        return;
      }
    }
    var1.CGLIB$CALLBACK_0 = (MethodInterceptor)((Callback[])var10000)[0];
  }
}
```
`CGLIB$BIND_CALLBACKS` 先从`CGLIB$THREAD_CALLBACKS`中`get`拦截对象，如果获取不到的话，再从`CGLIB$STATIC_CALLBACKS`来获取，如果也没有则认为该方法不需要代理。

那么拦截对象是如何设置到`CGLIB$THREAD_CALLBACKS` 或者 `CGLIB$STATIC_CALLBACKS`中的呢？

在Jdk动态代理中拦截对象是在实例化代理类时由构造函数传入的，在`cglib`中是调用`Enhancer`的`firstInstance`方法来生成代理类实例并设置拦截对象的。`firstInstance`的调用轨迹为：

1. Enhancer：`firstInstance`
2. Enhancer：`createUsingReflection`
3. Enhancer：`setThreadCallbacks`
4. Enhancer：`setCallbacksHelper`
5. `Target$$EnhancerByCGLIB$$788444a0` ： `CGLIB$SET_THREAD_CALLBACKS`

 在第5步，调用了代理类的`CGLIB$SET_THREAD_CALLBACKS`来完成拦截对象的注入。下面我们看一下`CGLIB$SET_THREAD_CALLBACKS`的反编译结果：

```java
public static void CGLIB$SET_THREAD_CALLBACKS(Callback[] var0) {
    CGLIB$THREAD_CALLBACKS.set(var0);
}
```

在`CGLIB$SET_THREAD_CALLBACKS`方法中调用了`CGLIB$THREAD_CALLBACKS`的`set`方法来保存拦截对象，在`CGLIB$BIND_CALLBACKS`方法中使用了`CGLIB$THREAD_CALLBACKS`的`get`方法来获取拦截对象，并保存到`CGLIB$CALLBACK_0`中。这样，在我们调用代理类的`f`方法时，就可以获取到我们设置的拦截对象，然后通过  *`tmp4_1.intercept(this, CGLIB$g$0$Method, CGLIB$emptyArgs, CGLIB$g$0$Proxy)`*  来实现代理。这里来解释一下`intercept`方法的参数含义：

@para1 **obj** ：代理对象本身

@para2 **method** ： 被拦截的方法对象

@para3 **args**：方法调用入参

@para4 **proxy**：用于调用被拦截方法的方法代理对象

这里会有一个疑问，为什么不直接反射调用代理类生成的（`CGLIB$f$0`）来间接调用目标类的被拦截方法，而使用`proxy`的`invokeSuper`方法呢？这里就涉及到了另外一个点— **FastClass** 。



# 三、FastClass

Jdk动态代理的拦截对象是通过反射的机制来调用被拦截方法的，**反射的效率比较低**，所以cglib采用了**FastClass**的机制来实现对被拦截方法的调用。**FastClass机制就是<u>对一个类的方法建立索引，通过索引来直接调用相应的方法</u>**。

在  `Target$$EnhancerByCGLIB$$cdd37a52` 类中的，静态代码块中，有如下一段

```java
Class var0 = Class.forName("com.noob.storage.aop.cglib.Target$$EnhancerByCGLIB$$cdd37a52");
Class var1;
Method[] var10000 = ReflectUtils.findMethods(new String[]{"equals", "(Ljava/lang/Object;)Z", "toString", "()Ljava/lang/String;", "hashCode", "()I", "clone", "()Ljava/lang/Object;"}, (var1 = Class.forName("java.lang.Object")).getDeclaredMethods());
CGLIB$f$0$Proxy = MethodProxy.create(var1, var0, "()V", "f", "CGLIB$f$0");
```

可以看到 `var0`  是cglib 生成的代理类，继续进入 create 方法查看

MethodProxy

```java
/**
 * For internal use by {@link Enhancer} only; see the {@link net.sf.cglib.reflect.FastMethod} class
 * for similar functionality.
 */
public static MethodProxy create(Class c1, Class c2, String desc, String name1, String name2) {
    MethodProxy proxy = new MethodProxy();
    proxy.sig1 = new Signature(name1, desc);
    proxy.sig2 = new Signature(name2, desc);
    proxy.createInfo = new CreateInfo(c1, c2);
    return proxy;
}
```

```java
public CreateInfo(Class c1, Class c2)
{
    this.c1 = c1;
    this.c2 = c2;
    AbstractClassGenerator fromEnhancer = AbstractClassGenerator.getCurrent();
    if (fromEnhancer != null) {
        namingPolicy = fromEnhancer.getNamingPolicy();
        strategy = fromEnhancer.getStrategy();
        attemptLoad = fromEnhancer.getAttemptLoad();
    }
}
```

这里可以看到代理类被赋值给了 CreateInfo 的第二个参数 c2

继续回到 MethodProxy 

```java
/**
 * Invoke the original method, on a different object of the same type.
 * @param obj the compatible object; recursion will result if you use the object passed as the first
 * argument to the MethodInterceptor (usually not what you want)
 * @param args the arguments passed to the intercepted method; you may substitute a different
 * argument array as long as the types are compatible
 * @see MethodInterceptor#intercept
 * @throws Throwable the bare exceptions thrown by the called method are passed through
 * without wrapping in an <code>InvocationTargetException</code>
 */
public Object invoke(Object obj, Object[] args) throws Throwable {
    try {
        init();
        FastClassInfo fci = fastClassInfo;
        return fci.f1.invoke(fci.i1, obj, args);
    } catch (InvocationTargetException e) {
        throw e.getTargetException();
    } catch (IllegalArgumentException e) {
        if (fastClassInfo.i1 < 0)
            throw new IllegalArgumentException("Protected method: " + sig1);
        throw e;
    }
}

/**
 * Invoke the original (super) method on the specified object.
 * @param obj the enhanced object, must be the object passed as the first
 * argument to the MethodInterceptor
 * @param args the arguments passed to the intercepted method; you may substitute a different
 * argument array as long as the types are compatible
 * @see MethodInterceptor#intercept
 * @throws Throwable the bare exceptions thrown by the called method are passed through
 * without wrapping in an <code>InvocationTargetException</code>
 */
public Object invokeSuper(Object obj, Object[] args) throws Throwable {
    try {
        init();
        FastClassInfo fci = fastClassInfo;
        return fci.f2.invoke(fci.i2, obj, args);
    } catch (InvocationTargetException e) {
        throw e.getTargetException();
    }
}
```

在init 方法内部，完成了 fci.f1  与 fci.f2 的初始化，具体代码如下

```java
private void init()
{
    /* 
     * Using a volatile invariant allows us to initialize the FastClass and
     * method index pairs atomically.
     * 
     * Double-checked locking is safe with volatile in Java 5.  Before 1.5 this 
     * code could allow fastClassInfo to be instantiated more than once, which
     * appears to be benign.
     */
    if (fastClassInfo == null)
    {
        synchronized (initLock)
        {
            if (fastClassInfo == null)
            {
                CreateInfo ci = createInfo;

                FastClassInfo fci = new FastClassInfo();
                fci.f1 = helper(ci, ci.c1);
                fci.f2 = helper(ci, ci.c2);
                fci.i1 = fci.f1.getIndex(sig1);
                fci.i2 = fci.f2.getIndex(sig2);
                fastClassInfo = fci;
                createInfo = null;
            }
        }
    }
}
```

```java
private static FastClass helper(CreateInfo ci, Class type) {
    FastClass.Generator g = new FastClass.Generator();
    g.setType(type);
    g.setClassLoader(ci.c2.getClassLoader());
    g.setNamingPolicy(ci.namingPolicy);
    g.setStrategy(ci.strategy);
    g.setAttemptLoad(ci.attemptLoad);
    return g.create();
}
```

所以可以看到，当调用`invokeSuper`方法时，实际上是调用代理类的`CGLIB$f$0`方法，`CGLIB$f$0`直接调用了目标类的`f`方法。所以，在第一节示例代码中我们使用`invokeSuper`方法来调用被拦截的目标类方法。