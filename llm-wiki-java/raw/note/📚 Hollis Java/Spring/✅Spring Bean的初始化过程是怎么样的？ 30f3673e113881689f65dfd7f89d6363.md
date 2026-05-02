# ✅Spring Bean的初始化过程是怎么样的？

所有者: junk01

# ✅Spring Bean的初始化过程是怎么样的？

> 原文：[https://www.yuque.com/hollis666/asgng6/zlvhpz](https://www.yuque.com/hollis666/asgng6/zlvhpz)
> 

# 典型回答

先看过上面这篇，我们就能知道，Spring的可以分为5个小的阶段：实例化、初始化、注册Destruction回调、Bean的正常使用以及Bean的销毁。

我们再把初始化的这个过程单独拿出来展开介绍一下。（本文代码基于 Spring 6.0版本）

首先先看一下初始化和实例化的区别是什么？

在Spring框架中，初始化和实例化是两个不同的概念：

**实例化（Instantiation）**：

- 实例化是创建对象的过程。在Spring中，这通常指的是通过调用类的构造器来创建Bean的实例。这是对象生命周期的开始阶段。对应doCreateBean中的createBeanInstance方法。

**初始化（Initialization）：**

- 初始化是在Bean实例创建后，进行一些设置或准备工作的过程。在Spring中，包括设置Bean的属性，调用各种前置&后置处理器。对应doCreateBean中的populateBean和initializeBean方法。

下面是SpringBean的实例化+初始化的完整过程：

### 实例化Bean

Spring容器在这一步创建Bean实例。其主要代码在AbstractAutowireCapableBeanFactory类中的createBeanInstance方法中实现：

其实就是先确保这个Bean对应的类已经被加载，然后确保它是public的，然后如果有工厂方法，则直接调用工厂方法创建这个Bean，如果没有的话就调用它的构造方法来创建这个Bean。

这里需要注意的是，在Spring的完整Bean创建和初始化流程中，容器会在调用createBeanInstance之前检查Bean定义的作用域。如果是Singleton，容器会在其内部单例缓存中查找现有实例。如果实例已存在，它将被重用；如果不存在，才会调用createBeanInstance来创建新的实例。

下一步就应该要到设置属性值了，但是在这之前还有一个重要的东西要讲，那就是三级解决循环依赖，在doCreateBean方法中：

这部分就是关于三级缓存解决循环依赖的内容。

### **设置属性值**

populateBean方法是Spring Bean生命周期中的一个关键部分，负责将属性值应用到新创建的Bean实例。它处理了自动装配、属性注入、依赖检查等多个方面。代码如下：

逻辑也比较清晰，就是把各种属性进行初始化。

### initializeBean方法

这个方法是初始化中的关键方法，后面要介绍的几个步骤就在这个方法中编排的：

### **检查Aware**

就是检查这个Bean是不是实现了BeanNameAware、BeanClassLoaderAware等这些Aware接口，Spring容器会调用它们的方法进行处理。

这些Aware接口提供了一种机制，使得Bean可以与Spring框架的内部组件交互，从而更灵活地利用Spring框架提供的功能。

- BeanNameAware: 通过这个接口，Bean可以获取到自己在Spring容器中的名字。这对于需要根据Bean的名称进行某些操作的场景很有用。
- BeanClassLoaderAware: 这个接口使Bean能够访问加载它的类加载器。这在需要进行类加载操作时特别有用，例如动态加载类。
- BeanFactoryAware：通过这个接口可以获取对 BeanFactory 的引用，获得对 BeanFactory 的访问权限

### 调用BeanPostProcessor的前置处理方法

BeanPostProcessor是Spring IOC容器给我们提供的一个扩展接口，他的主要作用主要是帮我们在Bean的初始化前后添加一些自己的逻辑处理，Spring内置了很多BeanPostProcessor，我们也可以定义一个或者多个 BeanPostProcessor 接口的实现，然后注册到容器中。

调用BeanPostProcessor的前置处理方法是在applyBeanPostProcessorsBeforeInitialization这个方法中实现的，代码如下：

其实就是遍历所有的BeanPostProcessor的实现，执行他的postProcessBeforeInitialization方法。

### **调用InitializingBean的afterPropertiesSet方法或自定义的初始化方法**

### **调用BeanPostProcessor的后置处理方法**

调用BeanPostProcessor的后置处理方法是在applyBeanPostProcessorsAfterInitialization这个方法中实现的，代码如下：

其实就是遍历所有的BeanPostProcessor的实现，执行他的postProcessAfterInitialization方法。

这里面需要我们关注的就是**AnnotationAwareAspectJAutoProxyCreator（继承自AspectJAwareAdvisorAutoProxyCreator），**他们也是BeanPostProcessor的实现，他之所以重要，是因为在他的postProcessAfterInitialization 后置处理方法。

在这里完成AOP的代理的创建。