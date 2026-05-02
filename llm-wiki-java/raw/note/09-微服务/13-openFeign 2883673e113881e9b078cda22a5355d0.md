# 13-openFeign

所有者: junk01

**1. 什么是Feign**

Feign是Netflix开发的声明式、模板化的HTTP客户端，其灵感来自Retrofit、JAXRS-2.0以及WebSocket。Feign可帮助我们更加便捷、优雅地调用HTTP API。Feign支持多种注解，例如Feign自带的注解或者JAX-RS注解等。

Spring Cloudopenfeign对Feign进行了增强，使其支持Spring MVC注解，另外还整合了Ribbon和Nacos，从而使得Feign的使用更加方便；编写调用接口+@FeignClient注解

**3.4 超时时间配置**

通过 Options 可以配置连接超时时间和读取超时时间，Options 的第一个参数是连接的超时时间（ms），默认值是 2s；第二个是请求处理的超时时间（ms），默认值是 5s。

补充说明：Feign的底层用的是Ribbon，但超时时间以Feign配置为准