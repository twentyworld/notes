## ApplicationContext && BeanFactory
---
### 定义
我们可以看[官方文档][2].

![img][1]
### 共同点
BeanFactory和ApplicationContext是Spring的两大核心接口，而其中ApplicationContext是BeanFactory的子接口。它们都可以当做Spring的容器，Spring容器是生成Bean实例的工厂，并管理容器中的Bean。在基于Spring的Java EE应用中，所有的组件都被当成Bean处理，包括数据源，Hibernate的SessionFactory、事务管理器等。
Spring容器最基本的接口就是BeanFactory。BeanFactory负责配置、创建、管理Bean，它有一个子接口ApplicationContext，也被称为Spring上下文，容器同时还管理着Bean和Bean之间的依赖关系。


### 区别
---
#### 特点

##### BeanFactory特点
BeanFactory常用的实现类是DefaultListableBeanFactory，调用者只需要使用getBean()方法就可以获得指定的引用，无须关心Bean的实例化过程。创建Spring容器的实例时，必须提供Spring容器管理的Bean的详细配置信息。Spring的配置信息通常采用XML配置文件来设置，因此，创建BeanFactory实例时，应该提供XML配置文件作为参数。XML配置文件通常使用Resource对象传入。

```
  Resource isr = new ClassPathResource("beans.xml");
  //Resource isr = new FileSystemResource("beans.xml");
  DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
  new XmlBeanDefinitionReader(beanFactory).loadBeanDefinition(isr);
```
##### ApplicationContext特点
ApplicationContext常用的实现类是FileSystemXmlApplicationContext、ClassPathXmlApplicationContext和AnnotationConfigApplicationContext。如果在Web应用中使用Spring容器，则通常使用XmlWebApplicationContext、AnnotationConfigApplicationContext两个实现类。如果需要使用多个XML配置文件创建Spring容器，可以使用FileSystemXmlApplicationContext或者是ClassPathXmlApplicationContext。

除了提供BeanFactory所支持的所有功能外，ApplicationContext还有额外的功能
- 默认初始化所有的Singleton，也可以通过配置取消预初始化。
- 资源访问，比如访问URL和文件。
- 事件机制。
- 同时加载多个配置文件。
- 以声明式方式启动并创建Spring容器。

---

### 总结
BeanFacotry是spring中比较原始的Factory。如XMLBeanFactory就是一种典型的BeanFactory。原始的BeanFactory无法支持spring的许多插件，如AOP功能、Web应用等。
ApplicationContext接口,它由BeanFactory接口派生而来，因而提供BeanFactory所有的功能。ApplicationContext以一种更向面向框架的方式工作以及对上下文进行分层和实现继承，ApplicationContext包还提供了以下的功能：
MessageSource, 提供国际化的消息访问
资源访问，如URL和文件
事件传播
载入多个（有继承关系）上下文 ，使得每一个上下文都专注于一个特定的层次，比如应用的web层

1. 利用MessageSource进行国际化
  BeanFactory是不支持国际化功能的，因为BeanFactory没有扩展Spring中MessageResource接口。相反，由于ApplicationContext扩展了MessageResource接口，因而具有消息处理的能力(i18N)，具体spring如何使用国际化，以后章节会详细描述。

2. 强大的事件机制(Event)
  基本上牵涉到事件(Event)方面的设计，就离不开观察者模式。这种模式在java开发中是比较常用的，又是比较重要的。
  ApplicationContext的事件机制主要通过ApplicationEvent和ApplicationListener这两个接口来提供的，和java swing中的事件机制一样。即当ApplicationContext中发布一个事件的时，所有扩展了ApplicationListener的Bean都将会接受到这个事件，并进行相应的处理。

  Spring提供了部分内置事件，主要有以下几种：

* ContextStartedEvent：生命周期 beans的启动信号
* ContextStoppedEvent: 生命周期 beans的停止信号
* ContextClosedEvent：ApplicationContext关闭事件，则context不能刷新和重启，从而所有的singleton bean全部销毁(因为singleton bean是存在容器缓存中的)

  虽然，spring提供了许多内置事件，但用户也可根据自己需要来扩展spriong中的事物。注意，要扩展的事件都要实现ApplicationEvent接口。

3. 底层资源的访问
ApplicationContext扩展了ResourceLoader(资源加载器)接口，从而可以用来加载多个Resource，而BeanFactory是没有扩展ResourceLoader

4. 对Web应用的支持
与BeanFactory通常以编程的方式被创建不同的是，ApplicationContext能以声明的方式创建，如使用ContextLoader。当然你也可以使用ApplicationContext的实现之一来以编程的方式创建ApplicationContext实例 。

ContextLoader有两个实现：ContextLoaderListener和ContextLoaderServlet。它们两个有着同样的功能，除了listener不能在Servlet 2.2兼容的容器中使用。自从Servelt 2.4规范，listener被要求在web应用启动后初始化。很多2.3兼容的容器已经实现了这个特性。使用哪一个取决于你自己，但是如果所有的条件都一样，你大概会更喜欢ContextLoaderListener；关于兼容方面的更多信息可以参照ContextLoaderServlet的JavaDoc。

  这个listener需要检查contextConfigLocation参数。如果不存在的话，它将默认使用/WEB-INF/applicationContext.xml。如果它存在，它就会用预先定义的分隔符（逗号，分号和空格）分开分割字符串，并将这些值作为应用上下文将要搜索的位置。ContextLoaderServlet可以用来替换ContextLoaderListener。这个servlet像listener那样使用contextConfigLocation参数。


5. 其它区别

1). BeanFactroy采用的是延迟加载形式来注入Bean的，即只有在使用到某个Bean时(调用getBean())，才对该Bean进行加载实例化，这样，我们就不能发现一些存在的Spring的配置问题。而ApplicationContext则相反，它是在容器启动时，一次性创建了所有的Bean。这样，在容器启动时，我们就可以发现Spring中存在的配置错误。

2). BeanFactory和ApplicationContext都支持BeanPostProcessor、BeanFactoryPostProcessor的使用，但两者之间的区别是：BeanFactory需要手动注册，而ApplicationContext则是自动注册




资料：
- [spring中的BeanFactory与ApplicationContext的作用和区别][3]







[1]:https://pic3.zhimg.com/80/v2-1006341abadfd3466b5b4587f349ab27_hd.jpg


[2]:https://docs.spring.io/spring/docs/5.0.6.BUILD-SNAPSHOT/spring-framework-reference/core.html#beans-beanfactory
[3]:https://blog.csdn.net/u011202334/article/details/51509235
