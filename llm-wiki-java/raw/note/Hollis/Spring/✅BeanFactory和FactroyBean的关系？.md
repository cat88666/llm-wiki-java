# ✅BeanFactory和FactroyBean的关系？

所有者: junk01

# ✅BeanFactory和FactroyBean的关系？

> 原文：[https://www.yuque.com/hollis666/asgng6/cnhqfg](https://www.yuque.com/hollis666/asgng6/cnhqfg)
> 

# 典型回答

FactoryBean和BeanFactory是Spring中的两个重要的概念。先看一下他们的类定义：

FactoryBean：

BeanFactory：

至少从代码上来看，这两个东西**都是接口（interface)**，然后**都是在org.springframework.beans.factory包**下面的。

网上有很多概念的解释，说明他俩的区别，但是很多人还是看不懂，下面是的解释我结合了具体的case，帮助大家更好地理解他们的作用，理解了各自的作用，那么区别自然也就理解了。

### BeanFactory

BeanFactory比较常用，名字也比较容易理解，就**是Bean工厂，他是整个Spring IoC容器的一部分，负责管理Bean的创建和生命周期。**

其中提供了一系列方法，可以让我们获取到具体的Bean实例。你可能没有直接用过BeanFactory，但是你肯定用过或者见过：

以上就是我们经常用的，在Spring的上下文中通过bean名称或者类型获取bean的方式，而这里的ApplicationContext，其实就是一种BeanFactory。这里面调用的getBean方法，就是上面我们在BeanFactory中看到的方法。

所以，**BeanFactory是Spring IoC容器的一个接口，用来获取Bean以及管理Bean的依赖注入和生命周期。**

### FactoryBean

**FactoryBean是一个接口，用于定义一个工厂Bean，它可以产生某种类型的对象。**当在Spring配置文件中定义一个Bean时，如果这个Bean实现了FactoryBean接口，那么Spring容器不直接返回这个Bean实例，而是返回FactoryBean#getObject()方法所返回的对象。

~~是不是还是听不懂？~~

那我给你举个具体的例子你就知道了。

Dubbo用过吧（没用过？那可能理解起来比较吃力，因为FactoryBean确实是在很多框架中用到的比较多，比如Kafka、dubbo等各种框架中都会用他来和Spring做集成）。

当我们想要在Dubbo中定义一个远程的提供者提供的的Bean的时候，可以用@DubboReference或者<dubbo:reference> 

而**这两种定义方式的最终实现都是一个Dubbo中的ReferenceBean ，它负责创建并管理远程服务代理对象。而这个ReferenceBean就是一个FactoryBean的实现。**

> 
> 

ReferenceBean的主要作用是创建并配置Dubbo服务的代理对象。这些代理对象允许客户端像调用本地方法一样调用远程服务。创建Dubbo服务代理通常涉及复杂的配置和初始化过程，包括网络通信设置、序列化配置等。通过ReferenceBean将这些复杂性封装起来，对于使用者来说，只需要通过简单的Spring配置即可使用服务。

ReferenceBean 实现了 FactoryBean 接口并实现了getObject方法。在getObject()方法中，ReferenceBean会给要调用的服务创建一个动态代理对象。这个代理对象负责与远程服务进行通信，封装了网络调用的细节，使得远程方法调用对于开发者来说是透明的。

通过 FactoryBean 实现，ReferenceBean 还可以延迟创建代理对象直到真正需要时，这样可以提升启动速度并减少资源消耗。此外，它还可以实现更复杂的加载策略和优化。

通过实现 FactoryBean，ReferenceBean 能够很好地与Spring框架集成。这意味着它可以利用Spring的依赖注入，生命周期管理等特性，并且能够被Spring容器所管理。

所以，**FactoryBean通常用于创建很复杂的对象，比如需要通过某种特定的创建过程才能得到的对象。例如，创建与JNDI资源的连接或与代理对象的创建。就如我们的Dubbo中的ReferenceBean。**

# 扩展知识

## ApplicationContext使用

我们知道，ApplicationContext作为BeanFactory的具体实现，他可以在Spring的的上下文中获取Bean，那么什么时候可以用到它呢？

一般来说，我们的Bean都是有Spring自动注入的，不太需要我们自己从上下文中获取，但是想让Spring帮忙注入，有一个前提，那就是必须被注入的Bean和注入的Bean都交给Spring托管，简单点说就是要有@Service和@Autowire一起用。

这样才能把HollisTestRepo注入到HollisTestService中，但是有的时候，我们的HollisTestService如果不是Spring托管的呢，比如是自己new的，那就不行了。

比较典型的是当有的时候我们使用充血模型，需要在里面去查询或者操作数据库的时候，就需要在这个模型中获取到对应的bean。那么就需要ApplicationContext了。

这里面SpringContextHolder的实现如下：