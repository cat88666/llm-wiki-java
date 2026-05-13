# ✅@PostConstruct、init-method和afterPropertiesSet执行顺序

所有者: junk01

# ✅@PostConstruct、init-method和afterPropertiesSet执行顺序

> 原文：[https://www.yuque.com/hollis666/asgng6/sgf2ipp88i6qk803](https://www.yuque.com/hollis666/asgng6/sgf2ipp88i6qk803)
> 

# 典型回答

在Spring框架中，使用@PostConstruct、自定义的init-method方法，和InitializingBean接口和afterPropertiesSet方法都是用于在Bean初始化阶段执行特定方法的方式。他们的执行顺序是：**构造函数>@PostConstruct > afterPropertiesSet > init-method**

- **@PostConstruct** 是javax.annotation 包中的注解(Spring Boot 3.0之后jakarta.annotation中，用于在构造函数执行完毕并且依赖注入完成后执行特定的初始化方法。标注在方法上，表示这个方法将在Bean初始化阶段被调用。
- **init-method** 是在Spring配置文件（如XML文件）中配置的一种方式。通过在Bean的配置中指定 init-method 属性，可以告诉Spring在Bean初始化完成后调用指定的初始化方法。如果不使用xml文件，也可以使用 @Bean 注解的 initMethod 属性来指定初始化方法。（下面的例子就是用的这种方式）
- **afterPropertiesSet** 是 Spring 的 InitializingBean 接口中的方法。如果一个 Bean 实现了 InitializingBean 接口，Spring 在初始化阶段会调用该接口的 afterPropertiesSet 方法。

Talk is Cheap，Show me the Code ，如下是一个示例：

启动Spring：

输出结果：·

所以，他们的执行顺序就是**构造函数>@PostConstruct > afterPropertiesSet > init-method**

# 扩展知识

## 顺序背后的原理

这个执行顺序，难道要死记硬背吗？那就太low了，其实这个执行顺序，如果大家看过下面这篇之后，就会理所应当的知道了。

在Bean的初始化过程中，在initializeBean方法中，会调用invokeInitMethods方法：

invokeInitMethods方法有两个if分支，第一个分支就是执行afterPropertiesSet方法的，第二个分支就是执行initMethod的，所以，一定是先执行afterPropertiesSet后执行initMethod。

那么，@PostConstruct的执行在哪个过程呢？

前面的initializeBean方法中，在调用invokeInitMethods之前，会调用applyBeanPostProcessorsBeforeInitialization，这个就是我们知道的调用BeanPostProcessor的前置处理方法。即调用postProcessBeforeInitialization

这里就需要登场一个InitDestroyAnnotationBeanPostProcessor了，看名字很容易知道，他就是在Bean初始化和销毁过程中的一个处理器。那么我们继续看postProcessBeforeInitialization方法：

这有一段熟悉的代码了，那就是invokeInitMethods，好像和我们要找的方法有关，看看metadata是咋来的，就看findLifecycleMetadata：

看到这里面的核心逻辑就是buildLifecycleMetadata，继续看buildLifecycleMetadata方法：

这里其实就是在寻找这个bean中有没有哪些方法上有initAnnotationType注解，如果有的话，那么就把他们添加到initMethods中，返回给前面的方法执行。

这里的initAnnotationTypes是一个集合：

看看他是从哪设置进来的就行了，直接找他在哪里被调用了，会发现在CommonAnnotationBeanPostProcessor中有调用：

因为我用的是Spring Boot 3.0，SpringBoot 2.0的代码长这样：

看到了吧，在这里设置了PostConstruct这个注解作为InitAnnotationType。

总结一下，在CommonAnnotationBeanPostProcessor初始化的时候，会把PostConstruct设置到initAnnotationTypes中，然后在InitDestroyAnnotationBeanPostProcessor的postProcessBeforeInitialization方法执行过程中，会检查这个Bean中有哪些initAnnotationType，把加了initAnnotationType注解的方法当做初始化方法进行调用。

所以，@PostConstruct 这个注解就在这个阶段被调用了。