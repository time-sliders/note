# web容器的加载过程

Web应用由Tomcat实例添加到Tomcat中，即由Tomcat管理一个新添加的Context容器。一个Web应用对应一个Context容器，也就是Servlet运行时的Servlet容器。 

1. 启动web项目后，web容器首先回去找web.xml文件，读取<context-param>和<listener>两个节点。
2. 容器会创建一个 ServletContext （ servlet 上下文），并将web.xml中的属性设置到Context容器中。整个 web 项目的所有部分都将共享这个上下文。
3. 容器将<context-param>转换为键值对，并交给 servletContext。因为listener, filter 等在初始化时会用到这些上下文中的信息，所以要先加载。 
4. 容器创建<listener>中的类实例，创建监听器。
5. 容器加载filter，创建过滤器， 要注意对应的filter-mapping一定要放在filter的后面。
6. 容器加载servlet，加载顺序按照 Load-on-startup 来执行

load- on-startup 元素在web应用启动的时候指定了servlet被加载的顺序，它的值必须是一个整数。
如果它的值是一个负整数或是这个元素不存在，那么容器会在该servlet被调用的时候，加载这个servlet。如果值是正整数或零，容器在配置的时候就加载并初始化这个servlet，容器必须保证值小的先被加载。如果值相等，容器可以自动选择先加载谁。

web.xml的完整加载顺序就是 ：ServletContext -> context-param -> listener-> filter -> servlet

不过有一点需要注意的是： **spring容器的加载要在servlet之后，因此在有些过滤器当中需要提前用到spring bean的时候，就需要改成 Listener 的方式 org.springframework.web.context.ContextLoaderListener。**

# Spring在web容器中的启动过程

1、对于一个web 应用，其部署在web 容器中，web 容器提供其一个全局的上下文环境，这个上下文就是 ServletContext ，ServletContext为其后面的spring IoC 容器提供宿主环境

2、在web.xml 中会提供有 contextLoaderListener。在web 容器启动时，会触发容器初始化事件，此时 contextLoaderListener 会监听到这个事件，其 contextInitialized 方法会被调用，在这个方法中，spring 会初始化一个启动上下文，这个上下文就被称为根上下文，即 WebApplicationContext ，这是一个 Interface，确切的说，其实际实现类是 XmlWebApplicaitonContext 。这个就是Spring 的Ioc 容器，其对应的Bean 定义的配置由web.xml 中的 context-param 标签指定。在这个Ioc 容器初始化完毕后，spring 以WebApplicationContext.ROOTWEBAPPLICATIONEXTATTRIBUTE 为属性key，将其存储到 servletContext 中，便于获取

3、contextLoaderListener 监听器初始化完毕后，开始初始化web.xml 中配置的servlet ，这个servlet 可以配置多个，以最常见的DispatcherServlet 为例，这个servlet 实际上是一个标准的前端控制器，用以转发、匹配、处理每个servlet 请求。DispatcherServlet 上下文在初始化的时候会建立自己的Ioc 上下文，用以持有springmvc 相关的bean。在建立DispatherSrvlet 自己的Ioc 上下文时，会利用 WebApplicationContext.ROOTWEBAPPLICATIONCONTEXTATTRIBUTE 先从ServletContext 中获取之前的根上下文（即 WebApplicationContext）作为自己上下文的parent 上下文，有了这个parent 上下文之后，再初始化自己持有的上下文。这个DispatcherServlet 初始化自己上下的工作在其 initStrategies 方法中可以看到，大概的工作就是初始化处理器映射、视图解析等，这个servlet 自己持有的上下文默认实现类也是 XmlWebApplicationContext。初始化完毕后，spring以与Servlet 的名字相关的属性为属性key，将其存到servletcontext 中，以便后续使用。这样每个Servlet 都持有自己的上下文，即拥有自己独立的bean 空间，同事各个servlet 共享相同的bean，即根上下文定义的那些bean。

介绍完了加载过程，在顺便了解下Servlet吧， 

1. 首先什么是servlet：

> servlet是sun公司为开发动态web而提供的一门技术，用户若想用发一个动态web资源(即开发一个Java程序向浏览器输出数据)，需要完成以下2个步骤： 
> 　　1、编写一个Java类，实现servlet接口。 
> 　　2、把开发好的Java类部署到web服务器中。 
> 　　按照一种约定俗成的称呼习惯，通常我们也把实现了servlet接口的java程序，称之为Servlet 

2.servlet的运行过程：

> 1.浏览器发出请求，被web容器获取到 
> 2.Web服务器首先检查是否已经装载并创建了该Servlet的实例对象。如果是，则直接执行第4步，否则，执行第3步。 
> 3.装载并创建该Servlet的一个实例对象，调用Servlet实例对象的init()方法。 
> 4.创建一个用于封装HTTP请求消息的HttpServletRequest对象和一个代表HTTP响应消息的HttpServletResponse对象，然后调用Servlet的service()方法并将请求和响应对象作为参数传递进去。 
> 5.WEB应用程序被停止或重新启动之前，Servlet引擎将卸载Servlet，并在卸载之前调用Servlet的destroy()方法。

**一般情况，servlet是在被请求的时候才去创建的，但是当添加时，就会在初始化的时候创建它，利用这点特性，我们可以初始化创建数据库等等。**

**使用servlet时，一般都是继承httpServlet，然后分别实现doGet或者doPost方法，<u>但是在这里面要注意的是，这servlet并不是线程安全的，多线程单实例执行的，当并发访问同一个资源的话，就有可能引发线程安全问题。</u>**