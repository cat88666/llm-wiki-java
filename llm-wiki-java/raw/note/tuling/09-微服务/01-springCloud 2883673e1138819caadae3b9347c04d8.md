# 01-springCloud

所有者: junk01

**SpringCloud与Dubbo的区别？**

Spring Cloud是⼀个微服务框架，提供了微服务领域中的很多功能组件，Dubbo⼀开始是⼀个RPC调⽤框 架，核⼼是解决服务调⽤间的问题，Spring Cloud是⼀个⼤⽽全的框架，Dubbo则更侧重于服务调⽤，所以Dubbo所提供的功能没有Spring Cloud全⾯，但是Dubbo的服务调⽤性能⽐Spring Cloud⾼，不过

Spring Cloud和Dubbo并不是对⽴的，是可以结合起来⼀起使⽤的。

**SpringCloud各组件功能？**

1. Eureka：注册中⼼，⽤来进⾏服务的⾃动注册和发现

2. Ribbon：负载均衡组件，⽤来在消费者调⽤服务时进⾏负载均衡

3. Feign：基于接⼝的申明式的服务调⽤客户端，让调⽤变得更简单

4. Hystrix：断路器，负责服务容错

5. Zuul：服务⽹关，可以进⾏服务路由、服务降级、负载均衡等

6. Nacos：分布式配置中⼼以及注册中⼼

7. Sentinel：服务的熔断降级，包括限流

8. Seata：分布式事务

9. Spring Cloud Config：分布式配置中⼼

10. Spring Cloud Bus：消息总线