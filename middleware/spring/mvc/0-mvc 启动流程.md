1. 初始化 Spring 应用 **Root WebApplicationContext**

2. 初始化  **WebApplicationContext for namespace 'mvc-servlet'**，并设置 parrent 为 **Root WebApplicationContext**

   这么做的原因是隔离：可以在Controller 层访问 Service，但是不允许在 Service 中注入 Controller。

3. Spring Mvc -> <mvc annotation-driven> 注解驱动解析，注入 RequestMappingHandlerMapping

4. RequestMappingHandlerMapping -> afterPropertySet -> 检测所有注解驱动的 Handler 并注册到当前 HandlerMapping

5. DispatchServlet -> onRefresh -> 扫描所有 HandlerMapping 实现类，并注入当前 servlet 中去

