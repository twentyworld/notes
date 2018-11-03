
## Spring-IOC


### IOC

### 基础知识
我写过一个通过ApplicationContext.xml来写的IOC， 代码在这里
https://github.com/twentyworld/learn/tree/master/Spring-Framework/SpringLearn

有一些很关键的类，你要知道他们做什么的
BeanFactory
ApplicationContext
ClassPathXmlApplicationContext
FactoryBean
---
##### 有哪些依赖注入方式
Setter 构造器
推荐使用哪种方式 为什么
可以结合使用吗。

---
##### 如何给Spring 容器提供配置元数据？

这里有三种重要的方法给Spring 容器提供配置元数据。

XML配置文件。
基于注解的配置。
基于java的配置。

#####  你怎样定义类的作用域？
不同配置元数据下的定义

##### Spring框架中的单例bean是线程安全的吗

#####  解释Spring框架中bean的生命周期

Spring容器 从XML 文件中读取bean的定义，并实例化bean。

Spring根据bean的定义填充所有的属性。

如果bean实现了BeanNameAware 接口，Spring 传递bean 的ID 到 setBeanName方法。

如果Bean 实现了 BeanFactoryAware 接口， Spring传递beanfactory 给setBeanFactory 方法。

如果有任何与bean相关联的BeanPostProcessors，Spring会在postProcesserBeforeInitialization()方法内调用它们。

如果bean实现IntializingBean了，调用它的afterPropertySet方法，如果bean声明了初始化方法，调用此初始化方法。

如果有BeanPostProcessors 和bean 关联，这些bean的postProcessAfterInitialization() 方法将被调用。

如果bean实现了 DisposableBean，它将调用destroy()方法。

> 注意，这里面有很多的类，这里面的类，都是你要知道的。这个才算是会Spring IOC.
> 我会问你这些东西的
> 如果可以的话，你可以写个例子，看再多，不如写个test。

##### 哪些是重要的bean生命周期方法？ 你能重载它们吗？

有两个重要的bean 生命周期方法，第一个是setup ， 它是在容器加载bean的时候被调用。第二个方法是 teardown  它是在容器卸载类的时候被调用。

The bean 标签有两个重要的属性（init-method和destroy-method）。用它们你可以自己定制初始化和注销方法。它们也有相应的注解（@PostConstruct和@PreDestroy）。

##### 在 Spring中如何注入一个java集合
有 哪些集合可以注入。

##### 什么是bean的自动装配
有五种自动装配的方式，可以用来指导Spring容器用自动装配方式来进行依赖注入

no：默认的方式是不进行自动装配，通过显式设置ref 属性来进行装配。

byName：通过参数名 自动装配，Spring容器在配置文件中发现bean的autowire属性被设置成byname，之后容器试图匹配、装配和该bean的属性具有相同名字的bean。

byType：通过参数类型自动装配，Spring容器在配置文件中发现bean的autowire属性被设置成byType，之后容器试图匹配、装配和该bean的属性具有相同类型的bean。如果有多个bean符合条件，则抛出错误。

constructor：这个方式类似于byType， 但是要提供给构造器参数，如果没有确定的带参数的构造器参数类型，将会抛出异常。

autodetect：首先尝试使用constructor来自动装配，如果无法工作，则使用byType方式。


>自动装配有哪些局限性？
>自动装配的局限性是：
>重写：你仍需用 <constructor-arg>和 <property> 配置来定义依赖，意味着总要重写自动装配。
>基本数据类型：你不能自动装配简单的属性，如基本数据类型，String字符串，和类。
>模糊特性：自动装配不如显式装配精确，如果有可能，建议使用显式装配。

##### 你可以在Spring中注入一个null 和一个空字符串吗？

可以。

##### 如何通过HibernateDaoSupport将Spring和Hibernate结合起来

参考文档
https://www.cnblogs.com/wang-meng/p/5701982.html
https://www.javacodegeeks.com/2014/05/spring-interview-questions-and-answers.html
https://www.cnblogs.com/liangyihui/p/5917773.html

### 面试中经常被问到的问题。
- Spring中Bean 的生命周期是怎么样的？
- Spring IOC是如何实现的，如何加载Bean，如何创建单例Bean？
- BeanFactory和ApplicationContext之间的关系？
- BeanFactory和FactoryBean之间的关系？
- Spring Bean有哪些扩展点？
- 谈谈你对spring IOC和DI的理解，它们有什么区别？
- IOC中用到的设计模式
- IOC的缺点

这几个问题的blog解决。
https://blog.csdn.net/qq_18860653/article/details/53218117
https://blog.csdn.net/hi_kevin/article/details/7325554
http://www.cnblogs.com/redcool/p/6413461.html
http://www.cnblogs.com/zhangminghui/p/4261377.html
https://www.cnblogs.com/wang-meng/p/5701982.html
https://blog.csdn.net/eluanshi12/article/details/80005687
https://blog.csdn.net/hla199106/article/details/47614527

参考阅读：
这个是最全的，包括IOC的特性，源码解读，还有高级特性。
https://blog.csdn.net/qq_34173549/article/details/79929071

---
有几个问题，需要你自己去做research
然后我听听结果
- Spring IOC 和工厂方法，我为什么不用工厂方法。
- Spring IOC 有没有什么缺点，你在实际开发中是怎么做的
    > intg test是怎么做的。
- Spring
