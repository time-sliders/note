## SecureClassLoader

SecureClassLoader **extends** ClassLoader

This class extends `ClassLoader` with additional **support for defining classes with an associated code source** and **permissions** which are retrieved by the system policy by default.

在 ClassLoader 构造器的基础上，添加了一些权限验证，如：

```java
protected SecureClassLoader(ClassLoader parent) {
    super(parent);
    // this is to make the stack depth consistent with 1.1
    SecurityManager security = System.getSecurityManager();
    if (security != null) {
        security.checkCreateClassLoader();
    }
    initialized = true;
}
```

## BuiltinClassLoader

BuiltinClassLoader  **extends** SecureClassLoader

The platform or application class loader. Resources loaded from modules defined to the boot class loader are also loaded via an instance of this ClassLoader type.

This ClassLoader supports loading of classes and resources from modules. Modules are defined to the ClassLoader by invoking the loadModule method. **Defining a module to this ClassLoader has the effect of making the types in the module visible**.

This ClassLoader also supports loading of classes and resources from a class path of URLs that are specified to the ClassLoader at construction time. The class path may expand`扩展` at runtime (the Class-Path attribute in JAR files or via instrumentation agents).

The ***delegation model*** used by this ClassLoader **differs to the regular delegation model. When requested to load a class then this ClassLoader first maps the class name to its package name. If there is a module defined to a BuiltinClassLoader containing this package then the class loader delegates directly to that class loader. If there isn't a module containing the package then it delegates the search to the parent class loader and if not found in the parent then it searches the class path. The main difference between this and the usual delegation model is that it allows the platform class loader to delegate to the application class loader, important with upgraded modules defined to the platform class loader**.

此类加载器使用的**委托模型与常规委托模型不同。当请求 loadClass 时，该 ClassLoader 首先将 class name映射到其package name。如果有一个模块定义为包含此 package 的 BuiltinClassLoader，那么将直接委托给该 ClassLoader。如果没有包含该 package 的 module，则它将搜索委托给 parrent，如果在父类中没有找到，则搜索 class path。这一模型与通常的委托模型的主要区别在于，它允许平台类加载器委托给应用程序类加载器，这一点对于定义到平台类加载器的升级模块很重要。**

#### SourceCode

##### Fields

```java
// parent ClassLoader
private final BuiltinClassLoader parent;

// the URL class path or null if there is no class path
private final URLClassPath ucp;

// maps package name to loaded module for modules in the boot layer
private static final Map<String, LoadedModule> packageToModule
		= new ConcurrentHashMap<>(SystemModules.PACKAGES_IN_BOOT_LAYER);
```

##### Constructor

```java
/**
 * Create a new instance.
 */
BuiltinClassLoader(String name, BuiltinClassLoader parent, URLClassPath ucp) {
  	// 由于 BootLoader 是 JVM 层级的类加载器，如果有被指定为 parent，则直接设置为 null
    // ensure getParent() returns null when the parent is the boot loader
    super(name, parent == null || parent == ClassLoaders.bootLoader() ? null : parent);

    this.parent = parent;
    this.ucp = ucp;

    this.nameToModule = new ConcurrentHashMap<>();
    this.moduleToReader = new ConcurrentHashMap<>();
}
```

##### loadClass

```java
/**
 * A variation of {@code loadCass} to load a class with the specified
 * binary name. This method returns {@code null} when the class is not
 * found.
 */
protected Class<?> loadClassOrNull(String cn, boolean resolve) {
    // 获取该 className 对应的 lock，避免在 parallel loader 时候存在重复 loader
    synchronized (getClassLoadingLock(cn)) { 

        // check if already loaded
        Class<?> c = findLoadedClass(cn);

        if (c == null) {

            // find the candidate module for this class
            LoadedModule loadedModule = findLoadedModule(cn);
            if (loadedModule != null) {

                // package is in a module
                BuiltinClassLoader loader = loadedModule.loader();
                if (loader == this) {
                    if (VM.isModuleSystemInited()) {
                        c = findClassInModuleOrNull(loadedModule, cn);
                    }
                } else {
                    // delegate to the other loader
                    c = loader.loadClassOrNull(cn);
                }

            } else {

                // check parent  委托给 parent
                if (parent != null) {
                    c = parent.loadClassOrNull(cn);
                }

                // check class path  parent没有则从ClassPath
                if (c == null && hasClassPath() && VM.isModuleSystemInited()) {
                    c = findClassOnClassPathOrNull(cn);
                }
            }

        }

        if (resolve && c != null)
            resolveClass(c);

        return c;
    }
}
```

这段代码可以看出，原有的双亲委派机制受到了模块化的影响，首先如果当前类已经加载了则直接返回，如果没加载，则根据名称找到对应的模块有没有加载，如果对应模块没有加载，则委派给父加载器去加载。如果对应模块已经加载了，则委派给对应模块的加载器去加载，这里需要注意下，**在模块里即使使用java.lang.Thread#setContextClassLoader方法改变当前上下文的类加载器，或者在模块里直接使用非当前模块的类加载器去加载当前模块里的类，最终使用的还是加载当前模块的类加载器。**

```java
/**
 * Finds the class with the specified binary name on the class path.
 *
 * @return the resulting Class or {@code null} if not found
 */
private Class<?> findClassOnClassPathOrNull(String cn) {
    String path = cn.replace('.', '/').concat(".class");
    if (System.getSecurityManager() == null) {
        Resource res = ucp.getResource(path, false);
        if (res != null) {
            try {
                return defineClass(cn, res);
            } catch (IOException ioe) {
                // TBD on how I/O errors should be propagated
            }
        }
        return null;
    } else {
        // avoid use of lambda here
        PrivilegedAction<Class<?>> pa = new PrivilegedAction<>() {
            public Class<?> run() {
                Resource res = ucp.getResource(path, false);
                if (res != null) {
                    try {
                        return defineClass(cn, res);
                    } catch (IOException ioe) {
                        // TBD on how I/O errors should be propagated
                    }
                }
                return null;
            }
        };
        return AccessController.doPrivileged(pa);
    }
}
```