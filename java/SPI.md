# 定义
全称为 (**Service Provider Interface**) ,是JDK内置的一种服务提供发现机制
一个服务(Service)通常指的是已知的接口或者抽象类，服务提供方就是对这个接口或者抽象类的实现，然后按照SPI 标准存放到资源路径**META-INF/services**目录下，文件的命名为该服务接口的全限定名。
许多开发框架都使用了Java的SPI机制，如java.sql.Driver的SPI实现（mysql驱动、oracle驱动等）、common-logging的日志接口实现、dubbo的扩展实现等等。

#使用方式
* 在 **META-INF/services/** 目录中创建以 Service 接口全限定名命名的文件，该文件内容为 Service 接口具体实现类的全限定名，文件编码必须为UTF-8。
* 使用`ServiceLoader.load(Class class);` 动态加载 Service 接口的实现类。
* 如 SPI 的实现类为 jar，则需要将其放在当前程序的 classpath 下。
* Service 的具体实现类必须有一个**不带参数的构造方法**。

# 出发点&优点
这种方式主要是针对不同的服务提供厂商，对不同场景的提供不同的解决方案制定的一套标准，举个简单的例子，如现在的JDK中有支持音乐播放，假设只支持mp3的播放，有些厂商想在这个基础之上支持mp4的播放，有的想支持mp5，而这些厂商都是第三方厂商，如果没有提供SPI这种实现标准，那就只有修改 JAVA 的源代码了，那这个弊端也是显而易见的，也就是不能够随着JDK的升级而升级现在的应用了，而有了SPI标准，SUN公司只需要提供一个播放接口，在实现播放的功能上通过ServiceLoad的方式加载服务，那么第三方只需要实现这个播放接口，再按SPI标准进行打包成jar，再放到classpath下面就OK了，没有一点代码的侵入性。

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
java.util
public final class **ServiceLoader** extends Object implements Iterable

A simple service-provider loading facility.

A service is a well-known set of interfaces and (usually abstract) classes. **A service provider is a specific implementation of a service**. The classes in a provider typically implement the interfaces and subclass the classes defined in the service itself. Service providers can be installed in an implementation of the Java platform in the form of extensions, that is, jar files placed into any of the usual extension directories. Providers can also be made available by adding them to the application's class path or by some other platform-specific means.

For the purpose of loading, a service is represented by a single type, that is, a single interface or abstract class. (A concrete class can be used, but this is not recommended.) A provider of a given service contains one or more concrete classes that extend this service type with data and code specific to the provider. The provider class is typically not the entire provider itself but rather a proxy which contains enough information to decide whether the provider is able to satisfy a particular request together with code that can create the actual provider on demand. The details of provider classes tend to be highly service-specific; no single class or interface could possibly unify them, so no such type is defined here. The only requirement enforced by this facility is that provider classes must have a zero-argument constructor so that they can be instantiated during loading.

A service provider is identified by placing a provider-configuration file in the resource directory META-INF/services. The file's name is the fully-qualified binary name of the service's type. The file contains a list of fully-qualified binary names of concrete provider classes, one per line. Space and tab characters surrounding each name, as well as blank lines, are ignored. The comment character is '#' ('\u0023', NUMBER SIGN); on each line all characters following the first comment character are ignored. The file must be encoded in UTF-8.

If a particular concrete provider class is named in more than one configuration file, or is named in the same configuration file more than once, then the duplicates are ignored. The configuration file naming a particular provider need not be in the same jar file or other distribution unit as the provider itself. The provider must be accessible from the same class loader that was initially queried to locate the configuration file; note that this is not necessarily the class loader from which the file was actually loaded.

Providers are located and instantiated lazily, that is, on demand. A service loader maintains a cache of the providers that have been loaded so far. Each invocation of the iterator method returns an iterator that first yields all of the elements of the cache, in instantiation order, and then lazily locates and instantiates any remaining providers, adding each one to the cache in turn. The cache can be cleared via the reload method.

Service loaders always execute in the security context of the caller. Trusted system code should typically invoke the methods in this class, and the methods of the iterators which they return, from within a privileged security context.

Instances of this class are not safe for use by multiple concurrent threads.
Unless otherwise specified, passing a null argument to any method in this class will cause a NullPointerException to be thrown.

Example Suppose we have a service type com.example.CodecSet which is intended to represent sets of encoder/decoder pairs for some protocol. In this case it is an abstract class with two abstract methods:
   public abstract Encoder getEncoder(String encodingName);
   public abstract Decoder getDecoder(String encodingName);

Each method returns an appropriate object or null if the provider does not support the given encoding. Typical providers support more than one encoding.

If com.example.impl.StandardCodecs is an implementation of the CodecSet service then its jar file also contains a file named
     META-INF/services/com.example.CodecSet
This file contains the single line:
     com.example.impl.StandardCodecs    # Standard codecs
The CodecSet class creates and saves a single service instance at initialization:
private static ServiceLoader<CodecSet> codecSetLoader = ServiceLoader.load(CodecSet.class);
To locate an encoder for a given encoding name it defines a static factory method which iterates through the known and available providers, returning only when it has located a suitable encoder or has run out of providers.

```java
   public static Encoder getEncoder(String encodingName) {
       for (CodecSet cp : codecSetLoader) {
           Encoder enc = cp.getEncoder(encodingName);
           if (enc != null)
               return enc;
       }
       return null;
   }
```

A `getDecoder` method is defined similarly.
Usage Note If the class path of a class loader that is used for provider loading includes remote network URLs then those URLs will be dereferenced in the process of searching for provider-configuration files.

A getDecoder method is defined similarly.
Usage Note If the class path of a class loader that is used for provider loading includes remote network URLs then those URLs will be dereferenced in the process of searching for provider-configuration files.

This activity is normal, although it may cause puzzling entries to be created in web-server logs. If a web server is not configured correctly, however, then this activity may cause the provider-loading algorithm to fail spuriously.

A web server should return an HTTP 404 (Not Found) response when a requested resource does not exist. Sometimes, however, web servers are erroneously configured to return an HTTP 200 (OK) response along with a helpful HTML error page in such cases. This will cause a ServiceConfigurationError to be thrown when this class attempts to parse the HTML page as a provider-configuration file. The best solution to this problem is to fix the misconfigured web server to return the correct response code (HTTP 404) along with the HTML error page.

Since:
1.6

Type parameters:
<S> - The type of the service to be loaded by this loader