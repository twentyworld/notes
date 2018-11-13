

## 一些关键字

### BeanPostProcessor
如上接口声明所示，BeanPostProcessor接口有两个回调方法。当一个BeanPostProcessor的实现类注册到Spring IOC容器后，对于该Spring IOC容器所创建的每个bean实例在初始化方法（如afterPropertiesSet和任意已声明的init方法）调用前，将会调用BeanPostProcessor中的postProcessBeforeInitialization方法，而在bean实例初始化方法调用完成后，则会调用BeanPostProcessor中的postProcessAfterInitialization方法，整个调用顺序可以简单示意如下

>
--> Spring IOC容器实例化Bean
--> 调用BeanPostProcessor的postProcessBeforeInitialization方法
--> 调用bean实例的初始化方法
--> 调用BeanPostProcessor的postProcessAfterInitialization方法

BeanPostProcessor是容器绑定的，即BeanPostProcessor只能对跟它属于同一个bean容器中的bean进行回调，即BeanPostProcessor不能对属于它父容器或子容器中的bean进行回调。

可以看到，Spring容器通过BeanPostProcessor给了我们一个机会对Spring管理的bean进行再加工。比如：我们可以修改bean的属性，可以给bean生成一个动态代理实例等等。一些Spring AOP的底层处理也是通过实现BeanPostProcessor来执行代理包装逻辑的。

具体可以参考博客:
- [Spring探秘|妙用BeanPostProcessor][1]
- [Spring（11）——可用来修改bean对象的BeanPostProcessor][2]

---

### BeanFactoryPostProcessor

BeanFactoryPostProcessor的核心作用是在bean在实例化前更改bean的配置信息。
这里我们可以从类的注释中清晰的发现：
```Java

/**
 * Allows for custom modification of an application context's bean definitions,
 * adapting the bean property values of the context's underlying bean factory.
 *
 * <p>Application contexts can auto-detect BeanFactoryPostProcessor beans in
 * their bean definitions and apply them before any other beans get created.
 *
 * <p>Useful for custom config files targeted at system administrators that
 * override bean properties configured in the application context.
 *
 * <p>See PropertyResourceConfigurer and its concrete implementations
 * for out-of-the-box solutions that address such configuration needs.
 *
 * <p>A BeanFactoryPostProcessor may interact with and modify bean
 * definitions, but never bean instances. Doing so may cause premature bean
 * instantiation, violating the container and causing unintended side-effects.
 * If bean instance interaction is required, consider implementing
 * {@link BeanPostProcessor} instead.
 */
@FunctionalInterface
public interface BeanFactoryPostProcessor {

	/**
	 * Modify the application context's internal bean factory after its standard
	 * initialization. All bean definitions will have been loaded, but no beans
	 * will have been instantiated yet. This allows for overriding or adding
	 * properties even to eager-initializing beans.
	 * @param beanFactory the bean factory used by the application context
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}

```
有几点比较关键：
- Application contexts can auto-detect BeanFactoryPostProcessor beans in their bean definitions and apply them before any other beans get created.
- A BeanFactoryPostProcessor may interact with and modify bean definitions, but never bean instances. Doing so may cause premature bean instantiation, violating the container and causing unintended side-effects. If bean instance interaction is required, consider implementing BeanPostProcessor instead.
- All bean definitions will have been loaded, but no beans will have been instantiated yet. This allows for overriding or adding properties even to eager-initializing beans.

但是有一些副作用，可以参考博客：
- [使用BeanFactoryPostProcessor——这种姿势不要用][3]


---

### 容器初始化和销毁Bean
- 第一种通过@PostConstruct和@PreDestroy方法实现初始化和销毁bean之前进行的操作
- 第二种是通过在xml中定义init-method和destory-method方法
- 第三种是通过bean实现InitializingBean和DisposableBean接口

他们是三种不同的方式去配置初始化Bean的方式。
也就是通过annotation，xml，config的方式。
---

### @Required
Spring依赖检查bean配置文件用于确定的特定类型(基本，集合或对象)的所有属性被设置。在大多数情况下，你只需要确保特定属性已经设置但不是所有属性。 对于这种情况，你需要@Required 注解。

简单地套用@Required注解不会强制执行该属性的检查，还需要注册一个RequiredAnnotationBeanPostProcessor以了解在bean配置文件@Required注解。
RequiredAnnotationBeanPostProcessor可以用两种方式来启用。

- 使用 `<context:annotation-config/>`隐式地注册RequiredAnnotationBeanPostProcessor， 使@Required注解生效
  ```XML
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:context="http://www.springframework.org/schema/context"
      xsi:schemaLocation="http://www.springframework.org/schema/beans
      http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
      http://www.springframework.org/schema/context
      http://www.springframework.org/schema/context/spring-context.xsd"
           default-lazy-init="true">

        <context:annotation-config/>
        ...
    </beans>
  ```
- 直接声明RequiredAnnotationBeanPostProcessor。
  ```XML
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:context="http://www.springframework.org/schema/context"
      xsi:schemaLocation="http://www.springframework.org/schema/beans
      http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
      http://www.springframework.org/schema/context
      http://www.springframework.org/schema/context/spring-context.xsd"
           default-lazy-init="true">

           <bean class="org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor"/>
        ...
    </beans>
  ```



---

### Spring框架中都用到了哪些设计模式？

代理模式：在AOP和remoting中被用的比较多。
单例模式：在spring配置文件中定义的bean默认为单例模式。
模板方法模式：用来解决代码重复的问题。
前端控制器模式：Spring提供了DispatcherServlet来对请求进行分发。
依赖注入模式：贯穿于BeanFactory / ApplicationContext接口的核心理念。
工厂模式：BeanFactory用来创建对象的实例。
















[1]:[https://www.jianshu.com/p/1417eefd2ab1]
[2]:[https://blog.csdn.net/elim168/article/details/76146351]
[3]:[https://www.jianshu.com/p/3d099ea43b0e]
