# SourceCode

## Fields

```java
// The parent class loader for delegation
// Note: VM hardcoded the offset of this field, thus all new fields
// must be added *after* it.
private final ClassLoader parent;
// class loader name
private final String name;
```

####  loadClass

```java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
								// 委派模式
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);// 刚方法由子类实现

                // this is the defining class loader; record the stats
                PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

# Javadoc

A class loader is an object that is responsible for`负责` **loading classes**. The class `ClassLoader` is an abstract class. Given the binary name of a class, a class loader should attempt to locate or generate data that constitutes`组成` a definition for the class. A typical strategy is to transform the name into a file name and then read a "class file" of that name from a file system.

<u>Every Class object contains a reference to the ClassLoader that defined it.</u>

<u>Class objects for **array classes** are not created by class loaders, but are created automatically as required by the Java runtime. The class loader for an array class, as returned by Class.getClassLoader() is the same as the class loader for its element type; if the element type is a primitive type, then the array class has no class loader.</u>

Applications implement subclasses of `ClassLoader` in order to extend the manner in which the Java virtual machine dynamically loads classes.

Class loaders may typically be used by security managers to indicate security domains.

In addition to loading classes, a class loader is also responsible for locating resources. A resource is some data (a ".class" file, configuration data, or an image for example) that is identified with an abstract '/'-separated path name. Resources are typically packaged with an application or library so that they can be located by code in the application or library. In some cases, the resources are included so that they can be located by other libraries.

<u>The `ClassLoader` class uses a **delegation model** to search for classes and resources. Each instance of `ClassLoader` has an associated **parent class loader**. When requested to find a class or resource, a `ClassLoader` instance will usually delegate the search for the class or resource to its parent class loader before attempting to find the class or resource itself. </u>

**委派模式的作用是，避免 application class 覆盖 jdk class**

Class loaders that support concurrent loading of classes are known as **parallel capable`可并行` class loaders** and are required to register themselves at their class initialization time by invoking the `ClassLoader.registerAsParallelCapable` method. Note that the `ClassLoader` class is registered as parallel capable by default. However, its subclasses still need to register themselves if they are parallel capable. In environments in which the delegation model is not strictly hierarchical`严格分级`, class loaders need to be parallel capable, otherwise class loading can lead to deadlocks because the **loader lock** is held for the duration of`在持续的时间内` the class loading process (see `loadClass` methods).

## Run-time Built-in Class Loaders
The Java run-time has the following built-in class loaders:

- **Bootstrap class loader**. It is the <u>virtual machine's built-in class loader</u>, typically represented as **null**, and does **not have a parent**.

- **Platform class loader**. All platform classes are visible to the platform class loader that can be used as the parent of a `ClassLoader` instance. **Platform classes include Java SE platform APIs, their implementation classes and JDK-specific run-time classes that are defined by the platform class loader or its ancestors.**
  To allow for upgrading/overriding of modules defined to the platform class loader, and where upgraded modules read modules defined to class loaders other than`除了…以外` the platform class loader and its ancestors`祖先`, then the platform class loader may have to delegate to other class loaders, the application class loader for example. In other words, classes in named modules defined to class loaders other than the platform class loader and its ancestors may be visible to the platform class loader.

- **System class loader**. It is also known as *application class loader* and is distinct from`不同于` the platform class loader. **The system class loader is typically used to define classes on the application class path, module path, and JDK-specific tools. The platform class loader is a parent or an ancestor of the system class loader that all platform classes are visible to it.**

Normally, the Java virtual machine loads classes from the local file system in a platform-dependent manner. However, some classes may not originate`来源于` from a file; they may originate from other sources, such as the network, or they could be constructed by an application. The method `defineClass` converts an array of bytes into an instance of class Class. Instances of this newly defined class can be created using `Class.newInstance`.
The methods and constructors of objects created by a class loader may reference other classes. To determine the class(es) referred to, the Java virtual machine invokes the `loadClass` method of the class loader that originally created the class`最初创建的类`.
For example, an application could create a network class loader to download class files from a server. Sample code might look like:

```java
     ClassLoader loader = new NetworkClassLoader(host, port);
     Object main = loader.loadClass("Main", true).newInstance();
          . . .
```

The network class loader subclass must define the methods `findClass` and `loadClassData` to load a class from the network. Once it has downloaded the bytes that make up the class, it should use the method `defineClass` to create a class instance. A sample implementation is:

```java
       class NetworkClassLoader extends ClassLoader {
           String host;
           int port;
           public Class findClass(String name) {
               byte[] b = loadClassData(name);
               return defineClass(name, b, 0, b.length);
           }
           private byte[] loadClassData(String name) {
               // load the class data from the connection
                . . .
           }
       }
```

**Binary names**
Any class name provided as a String parameter to methods in ClassLoader must be a binary name as defined by *The Java™ Language Specification*.
Examples of valid class names include:

```java
     "java.lang.String"
     "javax.swing.JSpinner$DefaultEditor"
     "java.security.KeyStore$Builder$FileBuilder$1"
     "java.net.URLClassLoader$3$1"
```

Any package name provided as a String parameter to methods in ClassLoader must be either the empty string (denoting an unnamed package) or a fully qualified name as defined by *The Java™ Language Specification*.