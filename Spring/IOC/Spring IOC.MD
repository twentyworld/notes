Spring IOC
---
 IoC is also known as dependency injection (DI). It is a process whereby objects define their dependencies, that is, the other objects they work with, only through constructor arguments, arguments to a factory method, or properties that are set on the object instance after it is constructed or returned from a factory method. The container then injects those dependencies when it creates the bean. This process is fundamentally the inverse, hence the name Inversion of Control (IoC), of the bean itself controlling the instantiation or location of its dependencies by using direct construction of classes, or a mechanism such as the Service Locator pattern.

## 1. IOC/DI
IoC 容器：最主要是完成了完成对象的创建和依赖的管理注入等等。对象和对象关系怎么表示:可以用 xml ， properties 文件等语义化配置文件表示。描述对象关系的文件存放在哪里:可能是 classpath ， filesystem ，或者是 URL 网络资源， servletContext 等。

## 2. 体系结构

### 2.1 BeanFactory
![UML][1]

**BeanFactory**:作为最顶层的一个接口类，它定义了IOC容器的基本功能规范。有三个子类：ListableBeanFactory、HierarchicalBeanFactory 和AutowireCapableBeanFactory。但是从上图中我们可以发现最终的默认实现类是 DefaultListableBeanFactory，他实现了所有的接口。那为何要定义这么多层次的接口呢？查阅这些接口的源码和说明发现，每个接口都有他使用的场合，它主要是为了区分在 Spring 内部在操作过程中对象的传递和转化过程中，对对象的数据访问所做的限制。

例如 ListableBeanFactory 接口表示这些 Bean 是可列表的，而 HierarchicalBeanFactory 表示的是这些 Bean 是有继承关系的，也就是每个Bean 有可能有父 Bean。AutowireCapableBeanFactory 接口定义 Bean 的自动装配规则。这四个接口共同定义了 Bean 的集合、Bean 之间的关系、以及 Bean 行为.

在BeanFactory里只对IOC容器的基本行为作了定义，根本不关心你的bean是如何定义怎样加载的。正如我们只关心工厂里得到什么的产品对象，至于工厂是怎么生产这些对象的，这个基本的接口不关心。

```
public interface BeanFactory {

	Object getBean(String name) throws BeansException;
	<T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException;
	Object getBean(String name, Object... args) throws BeansException;
	<T> T getBean(Class<T> requiredType) throws BeansException;
	boolean containsBean(String name);
	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;
	boolean isTypeMatch(String name, @Nullable Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

	@Nullable
	Class<?> getType(String name) throws NoSuchBeanDefinitionException;
	String[] getAliases(String name);
}
```

而要知道工厂是如何产生对象的，我们需要看具体的IOC容器实现，spring提供了许多IOC容器的实现。

### 2.2 BeanDefinition
SpringIOC容器管理了我们定义的各种Bean对象及其相互的关系，Bean对象在Spring实现中是以BeanDefinition来描述的. 其继承体系如下![][2]：


## 3. IOC容器初始化
IoC容器的初始化包括BeanDefinition的Resource定位、载入和注册这三个基本的过程。
在这里，以ApplciationContext为例。
![][3]:
ApplicationContext允许上下文嵌套，通过保持父上下文可以维持一个上下文体系。对于bean的查找可以在这个上下文体系中发生，首先检查当前上下文，其次是父上下文，逐级向上，这样为不同的Spring应用提供了一个共享的bean定义环境。

### 3.1 XmlBeanFactory的容器创建过程。
![][4]

XmlBeanFactory源码。

```
public class XmlBeanFactory extends DefaultListableBeanFactory {
	private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);

	public XmlBeanFactory(Resource resource) throws BeansException {
		this(resource, null);
	}
	public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
		super(parentBeanFactory);
		this.reader.loadBeanDefinitions(resource);
	}
}
```

### 3.2 FileSystemXmlApplicationContext(xmlPath)的IOC容器流程。
我们调用FileSystemXmlApplicationContext：
```
ApplicationContext =new FileSystemXmlApplicationContext(xmlPath);
```
我们调用FileSystemXmlApplicationContext的构造方法。
```
public class FileSystemXmlApplicationContext extends AbstractXmlApplicationContext {
	public FileSystemXmlApplicationContext() {
	}

	public FileSystemXmlApplicationContext(ApplicationContext parent) {
		super(parent);
	}

	public FileSystemXmlApplicationContext(String configLocation) throws BeansException {
		this(new String[] {configLocation}, true, null);
	}


  public FileSystemXmlApplicationContext(String... configLocations) throws BeansException {
		this(configLocations, true, null);
	}

	public FileSystemXmlApplicationContext(String[] configLocations, ApplicationContext parent) throws BeansException {
		this(configLocations, true, parent);
	}

	public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh) throws BeansException {
		this(configLocations, refresh, null);
	}

	public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {
		super(parent);
		setConfigLocations(configLocations);
		if (refresh) refresh();
	}
	protected Resource getResourceByPath(String path) {
		if (path.startsWith("/")) {
			path = path.substring(1);
		}
		return new FileSystemResource(path);
	}
}
```
通过分析FileSystemXmlApplicationContext的源代码可以知道，在创建FileSystemXmlApplicationContext容器时，构造方法做以下两项重要工作：

- 调用父类容器的构造方法(super(parent)方法)为容器设置好Bean资源加载器。
- 再调用父类AbstractRefreshableConfigApplicationContext的setConfigLocations(configLocations)方法设置Bean定义资源文件的定位路径。

#### 3.2.1 为容器设置Bean资源加载器
通过追踪FileSystemXmlApplicationContext的继承体系，发现AbstractApplicationContext中初始化IoC容器所做的主要源码如下：
```
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext {
    //静态初始化块，在整个容器创建过程中只执行一次  
    static {
  		// Eagerly load the ContextClosedEvent class to avoid weird classloader issues
  		// on application shutdown in WebLogic 8.1. (Reported by Dustin Woods.)
  		ContextClosedEvent.class.getName();
  	}
    //AbstractApplicationContext构造方法中调用PathMatchingResourcePatternResolver的构造方法创建Spring资源加载器
  	public AbstractApplicationContext() {
  		this.resourcePatternResolver = getResourcePatternResolver();
  	}
    //FileSystemXmlApplicationContext调用父类构造方法调用的就是该方法  
  	public AbstractApplicationContext(@Nullable ApplicationContext parent) {
  		this();
  		setParent(parent);
  	}
  //获取一个Spring Source的加载器用于读入Spring Bean定义资源文件  
    protected ResourcePatternResolver getResourcePatternResolver() {
      return new PathMatchingResourcePatternResolver(this);
    }

    @Override
    public void setParent(@Nullable ApplicationContext parent) {
      this.parent = parent;
      if (parent != null) {
        Environment parentEnvironment = parent.getEnvironment();
        if (parentEnvironment instanceof ConfigurableEnvironment) getEnvironment().merge((ConfigurableEnvironment) parentEnvironment);
      }
    }
}
```

#### 3.2.2 设置Bean定义资源文件的定位路径
在设置容器的资源加载器之后，接下来FileSystemXmlApplicationContet执行setConfigLocations方法通过调用其父类AbstractRefreshableConfigApplicationContext的方法进行对Bean定义资源文件的定位，该方法的源码如下：
```
public void setConfigLocations(@Nullable String... locations) {
  if (locations != null) {
    Assert.noNullElements(locations, "Config locations must not be null");
    this.configLocations = new String[locations.length];
    for (int i = 0; i < locations.length; i++) {
      this.configLocations[i] = resolvePath(locations[i]).trim();
    }
  }
  else {
    this.configLocations = null;
  }
}

  protected String resolvePath(String path) {
    return getEnvironment().resolveRequiredPlaceholders(path);
  }

  public ConfigurableEnvironment getEnvironment() {
    if (this.environment == null) {
      this.environment = createEnvironment();
    }
    return this.environment;
  }
  protected ConfigurableEnvironment createEnvironment() {
    return new StandardEnvironment();
  }

}
```
至此，Spring IoC容器在初始化时将配置的Bean定义资源文件定位为Spring封装的Resource。

#### 3.2.3 refresh函数载入Bean定义过程
早前放的图可知，这里面的继承顺序：
AbstractApplicationContext<-AbstractRefreshableApplicationContext<-AbstractRefreshableConfigApplicationContext
  <-AbstractXmlApplicationContext<-FileSystemXmlApplicationContext
```
public void refresh() throws BeansException, IllegalStateException {
  synchronized (this.startupShutdownMonitor) {
    // Prepare this context for refreshing.
    prepareRefresh();

    // Tell the subclass to refresh the internal bean factory.
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

    // Prepare the bean factory for use in this context.
    prepareBeanFactory(beanFactory);

    try {
      // Allows post-processing of the bean factory in context subclasses.
      postProcessBeanFactory(beanFactory);
      // Invoke factory processors registered as beans in the context.
      invokeBeanFactoryPostProcessors(beanFactory);
      // Register bean processors that intercept bean creation.
      registerBeanPostProcessors(beanFactory);
      // Initialize message source for this context.
      initMessageSource();
      // Initialize event multicaster for this context.
      initApplicationEventMulticaster();
      // Initialize other special beans in specific context subclasses.
      onRefresh();
      // Check for listener beans and register them.
      registerListeners();
      // Instantiate all remaining (non-lazy-init) singletons.
      finishBeanFactoryInitialization(beanFactory);
      // Last step: publish corresponding event.
      finishRefresh();
    }
    catch (BeansException ex) {
      if (logger.isWarnEnabled())
        logger.warn("Exception encountered during context initialization - " + "cancelling refresh attempt: " + ex);
      // Destroy already created singletons to avoid dangling resources.
      destroyBeans();
      // Reset 'active' flag.
      cancelRefresh(ex);
      // Propagate exception to caller.
      throw ex;
    }
    finally {
      // Reset common introspection caches in Spring's core, since we
      // might not ever need metadata for singleton beans anymore...
      resetCommonCaches();
    }
  }
}

protected void prepareRefresh() {
  this.startupDate = System.currentTimeMillis();
  this.closed.set(false);
  this.active.set(true);
  if (logger.isInfoEnabled())logger.info("Refreshing " + this);

  // Initialize any placeholder property sources in the context environment
  initPropertySources();
  // Validate that all properties marked as required are resolvable
  // see ConfigurablePropertyResolver#setRequiredProperties
  getEnvironment().validateRequiredProperties();
  // Allow for the collection of early ApplicationEvents,
  // to be published once the multicaster is available...
  this.earlyApplicationEvents = new LinkedHashSet<>();
}
```
Spring IoC容器对Bean定义资源的载入是从refresh()函数开始的，refresh()是一个模板方法，refresh()方法的作用是：在创建IoC容器前，如果已经有容器存在，则需要把已有的容器销毁和关闭，以保证在refresh之后使用的是新建立起来的IoC容器。refresh的作用类似于对IoC容器的重启，再新建立好的容器中对容器进行初始化，对Bean定义资源进行载入。

refresh()方法主要为IoC容器Bean的生命周期管理提供条件，Spring IoC容器载入Bean定义资源文件从其子类容器的refreshBeanFactory()方法启动，所以整个refresh()中“ConfigurableListableBeanFactory beanFactory =obtainFreshBeanFactory();”这句以后代码的都是注册容器的信息源和生命周期事件，载入过程就是从这句代码启动。

###### 3.2.3.1 obtainFreshBeanFactory()方法 刷新容器
**AbstractApplicationContext** 中实现。
```
//Tell the subclass to refresh the internal bean factory.
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
  //这里使用了委派设计模式，父类定义了抽象的refreshBeanFactory()方法，具体实现调用子类容器的refreshBeanFactory()方法
  refreshBeanFactory();
  ConfigurableListableBeanFactory beanFactory = getBeanFactory();
  if (logger.isDebugEnabled()) {
    logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
  }
  return beanFactory;
}
```

在 **AbstractRefreshableApplicationContext** 中有refreshBeanFactory().
其中有关闭原来的容器，重建容器。
```
/**
 * This implementation performs an actual refresh of this context's underlying
 * bean factory, shutting down the previous bean factory (if any) and
 * initializing a fresh bean factory for the next phase of the context's lifecycle.
 */
@Override
protected final void refreshBeanFactory() throws BeansException {
  if (hasBeanFactory()) {
    //Template method for destroying all beans that this context manages.
    destroyBeans();
    closeBeanFactory();
  }
  try {
    DefaultListableBeanFactory beanFactory = createBeanFactory();
    beanFactory.setSerializationId(getId());
    customizeBeanFactory(beanFactory);
    loadBeanDefinitions(beanFactory);
    synchronized (this.beanFactoryMonitor) {
      this.beanFactory = beanFactory;
    }
  }
  catch (IOException ex) {
    throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
  }
}

@Override
protected final void closeBeanFactory() {
  synchronized (this.beanFactoryMonitor) {
    if (this.beanFactory != null)
      this.beanFactory.setSerializationId(null);
      this.beanFactory = null;
  }
}
protected DefaultListableBeanFactory createBeanFactory() {
  return new DefaultListableBeanFactory(getInternalParentBeanFactory());
}
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
  if (this.allowBeanDefinitionOverriding != null) {
    beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
  }
  if (this.allowCircularReferences != null) {
    beanFactory.setAllowCircularReferences(this.allowCircularReferences);
  }
}
```
在AbstractXmlApplicationContext中调用loadBeanDefinitions()。加载bean。
```
//Loads the bean definitions via an XmlBeanDefinitionReader.
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
  // Create a new XmlBeanDefinitionReader for the given BeanFactory.
  XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

  // Configure the bean definition reader with this context's
  // resource loading environment.
  beanDefinitionReader.setEnvironment(this.getEnvironment());

  //祖先父类AbstractApplicationContext继承DefaultResourceLoader，因此，容器本身也是一个资源加载器  
  beanDefinitionReader.setResourceLoader(this);
   //为Bean读取器设置SAX xml解析器
  beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

  // Allow a subclass to provide custom initialization of the reader,
  // then proceed with actually loading the bean definitions.
  initBeanDefinitionReader(beanDefinitionReader);
  //Bean读取器真正实现加载的方法  
  loadBeanDefinitions(beanDefinitionReader);
}

//Load the bean definitions with the given XmlBeanDefinitionReader.
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
  Resource[] configResources = getConfigResources();
  if (configResources != null) {
    reader.loadBeanDefinitions(configResources);
  }
  //如果子类中获取的Bean定义资源定位为空，则获取FileSystemXmlApplicationContext构造方法中setConfigLocations方法设置的资源
  String[] configLocations = getConfigLocations();
  if (configLocations != null) {
    reader.loadBeanDefinitions(configLocations);
  }
}


//这里又使用了一个委托模式，调用子类的获取Bean定义资源定位的方法。
//该方法在ClassPathXmlApplicationContext中进行实现，的FileSystemXmlApplicationContext没有使用该方法。
@Nullable
protected Resource[] getConfigResources() {
  return null;
}

```
这里面调用了XmlBeanDefinitionReader..loadBeanDefinitions(configLocations);















































[1]:https://images0.cnblogs.com/blog/400827/201409/172219470349285.x-png
[2]:https://s26.postimg.org/5pisndljd/spring_beandefinition.png
[3]:https://s26.postimg.org/s2qjah4k9/classpathxmlapplicaitoncontext.png
[4]:https://s26.postimg.org/vmch0e9vt/xmlbean_Factory.png
