# service bean 的创建
spring-dubbo.xml 示例 

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
           xsi:schemaLocation="http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
           
        <dubbo:application name="tz-transfer" />
        <dubbo:registry server="zkRegistry" protocol="${dubbo.registry.protocol}" address="${dubbo.registry.address}" file="${dubbo.registry.file}" />
        <dubbo:protocol port="${dubbo.protocol.port}" />
        <dubbo:provider timeout="${dubbo.provider.timeout}" />
        <dubbo:consumer check="false" />
        
        <dubbo:service interface="com.xxx.xxxxFacade" ref="xxxxFacadeImpl" version="1.0" />
        <dubbo:reference id="xxxxFacade" interface="com.xxxx.facade.service.xxxxFacade" version="1.0" timeout="30000"/>
    </beans>

spring-dubbo.xml 是由spring所支持的自定义配置 spring.handler 配置文件中指定的处理器来解析的，从dubbo-config/dubbo-config-spring 模块中能看到该配置文件

    http\://dubbo.apache.org/schema/dubbo=org.apache.dubbo.config.spring.schema.DubboNamespaceHandler
    http\://code.alibabatech.com/schema/dubbo=org.apache.dubbo.config.spring.schema.DubboNamespaceHandler

打开  DubboNamespaceHandler 类

    registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
    
可以看到dubbo 是用 DubboBeanDefinitionParser 来解析dubbo.xml的 service 配置元素，打开 DubboBeanDefinitionParser，主要关注 parse 方法。将dubbo配置文件解析成 BeanDefinition 并注册到spring#ParserContext#BeanRegistry 中。

# service bean 暴露服务
打开 ServiceBean，该对象实现 InitialBean接口，在 afterPropertiesSet 方法内部实现数据初始化 + 服务暴露逻辑
在方法的最后，调用 export() 方法暴露服务，之后，通过 publishExportEvent() 方法，告诉注册中心服务暴露成功，可以供 consumer 消费，先看第一步，export() -> doExport() -> doExportUrls() -> doExportUrlsFor1Protocol(protocolConfig, registryURLs); -> Exporter<?> exporter = protocol.export(wrapperInvoker); -> DubboProtocol.export(...) -> openServer(url); -> createServer(url) -> Exchangers.bind(url, requestHandler); -> HeaderExchanger.bind() -> Transporters.bind() -> NettyTransporter.bind() -> NettyServer.doOpen() -> 此时一个基于 netty 的服务端口暴露成功

# 如何注册服务到 zookeeper ?
