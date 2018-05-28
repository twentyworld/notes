# Spring IOC
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

```Java
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
SpringIOC容器管理了我们定义的各种Bean对象及其相互的关系，Bean对象在Spring实现中是以BeanDefinition来描述的. 其继承体系如下

![][2]


## 3. IOC容器初始化
IoC容器的初始化包括BeanDefinition的Resource定位、载入和注册这三个基本的过程。
在这里，以ApplciationContext为例。

![][3]

ApplicationContext允许上下文嵌套，通过保持父上下文可以维持一个上下文体系。对于bean的查找可以在这个上下文体系中发生，首先检查当前上下文，其次是父上下文，逐级向上，这样为不同的Spring应用提供了一个共享的bean定义环境。

### 3.1 XmlBeanFactory的容器创建过程。
![][4]

XmlBeanFactory源码。

```Java
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
```Java
ApplicationContext =new FileSystemXmlApplicationContext(xmlPath);
```
我们调用FileSystemXmlApplicationContext的构造方法。
```Java
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

	public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, @Nullable ApplicationContext parent) throws BeansException {
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
他们都在最后通过调用到了：
```Java
FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
```
在对象的初始化过程中，调用refresh函数载入`beandefinition`，
通过分析FileSystemXmlApplicationContext的源代码可以知道，在创建FileSystemXmlApplicationContext容器时，构造方法做以下两项重要工作：

- 调用父类容器的构造方法(super(parent)方法)为容器设置好Bean资源加载器。
- 再调用父类AbstractRefreshableConfigApplicationContext的setConfigLocations(configLocations)方法设置Bean定义资源文件的定位路径。

#### 3.2.1 为容器设置Bean资源加载器
通过追踪FileSystemXmlApplicationContext的继承体系，发现AbstractApplicationContext中初始化IoC容器所做的主要源码如下：
```Java
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
        if (parentEnvironment instanceof ConfigurableEnvironment)
          getEnvironment().merge((ConfigurableEnvironment) parentEnvironment);
      }
    }
}
```

#### 3.2.2 设置Bean定义资源文件的定位路径
在设置容器的资源加载器之后，接下来FileSystemXmlApplicationContet执行setConfigLocations方法通过调用其父类AbstractRefreshableConfigApplicationContext的方法进行对Bean定义资源文件的定位，该方法的源码如下：
```Java
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
```Java
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
        logger.warn("Exception encountered during context initialization - "
          - "cancelling refresh attempt: " + ex);
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
```Java
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
```Java
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
```Java
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
这里面调用了XmlBeanDefinitionReader..loadBeanDefinitions(configLocations);由这里，引入了 BeanDefinition 读取Bean定义资源。

#### 3.2.4 BeanDefinition 读取Bean定义资源

##### 3.2.4.1 XmlBeanDefinitionReader 读取文件。
![][5]
在抽象类AbstractBeanDefinitionReader中定义了
loadBeanDefinitions(String... locations) -> loadBeanDefinitions(String location) ->
  loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) -> loadBeanDefinitions(resources)
   -> loadBeanDefinitions(resource) -> loadBeanDefinitions(new EncodedResource(resource)) -> doLoadBeanDefinitions(inputSource, encodedResource.getResource())的路径，找到了文件的路径。

```Java
public abstract class AbstractBeanDefinitionReader implements EnvironmentCapable, BeanDefinitionReader {

  public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
		Assert.notNull(locations, "Location array must not be null");
		int counter = 0;
		for (String location : locations) {
			counter += loadBeanDefinitions(location);
		}
		return counter;
	}

  public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(location, null);
	}
  public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
		}

		if (resourceLoader instanceof ResourcePatternResolver) {
			// Resource pattern matching available.
			try {
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
				int loadCount = loadBeanDefinitions(resources);
				if (actualResources != null) {
					for (Resource resource : resources) {
						actualResources.add(resource);
					}
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
				}
				return loadCount;
			}
			catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
			}
		}
		else {
			// Can only load single resources by absolute URL.
			Resource resource = resourceLoader.getResource(location);
			int loadCount = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
			}
			return loadCount;
		}
	}

}
```

##### 3.2.4.2 读取资源转换为Document

这里做了两件事情，第一解析xml文件，同时把bean注册到容器中。
 XmlBeanDefinitionReader类中的doLoadBeanDefinitions方法是从特定XML文件中实际载入Bean定义资源的方法，该方法在载入Bean定义资源之后将其转换为Document对象，接下来调用registerBeanDefinitions启动Spring IoC容器对Bean定义的解析过程.

```Java
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {
    protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
        throws BeanDefinitionStoreException {
      try {
        Document doc = doLoadDocument(inputSource, resource);
        return registerBeanDefinitions(doc, resource);
      }
      catch (Exception ex) {
        ...
      }
    }

    protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
  		return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
  				getValidationModeForResource(resource), isNamespaceAware());
  	}
}
```
##### 3.2.4.3 DocumentLoader解析xml文件。
这阶段，是由DocumentLoader的实现类DefaultDocumentLoader具体实现。

``` Java
public class DefaultDocumentLoader implements DocumentLoader {
  //使用标准的JAXP将载入的Bean定义资源转换成document对象  
	public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
			ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {

//创建文件解析器工厂  
		DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
		if (logger.isDebugEnabled()) {
			logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
		}
    //创建文档解析器  
		DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
    //解析Spring的Bean定义资源  
		return builder.parse(inputSource);
	}

  protected DocumentBuilderFactory createDocumentBuilderFactory(int validationMode, boolean namespaceAware)
			throws ParserConfigurationException {
//创建文档解析工厂  
		DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
		factory.setNamespaceAware(namespaceAware);
      //设置解析XML的校验  
		if (validationMode != XmlValidationModeDetector.VALIDATION_NONE) {
			factory.setValidating(true);
			if (validationMode == XmlValidationModeDetector.VALIDATION_XSD) {
				// Enforce namespace aware for XSD...
				factory.setNamespaceAware(true);
				try {
					factory.setAttribute(SCHEMA_LANGUAGE_ATTRIBUTE, XSD_SCHEMA_LANGUAGE);
				}
				catch (IllegalArgumentException ex) {
					ParserConfigurationException pcex = new ParserConfigurationException(
							"Unable to validate using XSD: Your JAXP provider [" + factory +
							"] does not support XML Schema. Are you running on Java 1.4 with Apache Crimson? " +
							"Upgrade to Apache Xerces (or Java 1.5) for full XSD support.");
					pcex.initCause(ex);
					throw pcex;
				}
			}
		}
}
```

##### 3.2.4.4 XmlBeanDefinitionReader解析载入的Bean定义资源文件

```Java
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {
    public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
      BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
      int countBefore = getRegistry().getBeanDefinitionCount();
      documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
      return getRegistry().getBeanDefinitionCount() - countBefore;
    }
    protected BeanDefinitionDocumentReader createBeanDefinitionDocumentReader() {
      return BeanDefinitionDocumentReader.class.cast(BeanUtils.instantiateClass(this.documentReaderClass));
    }
    private Class<?> documentReaderClass = DefaultBeanDefinitionDocumentReader.class;
}
```
类XmlBeanDefinitionReader中的registerBeanDefinitions方法调用了DefaultBeanDefinitionDocumentReader中的方法 doRegisterBeanDefinitions(Element root) 对documnet对象进行解析：
```Java
public class DefaultBeanDefinitionDocumentReader implements BeanDefinitionDocumentReader {


  public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    //获得XML描述符
    this.readerContext = readerContext;
    logger.debug("Loading bean definitions");
    //获得Document的根元素  
    Element root = doc.getDocumentElement();

    doRegisterBeanDefinitions(root);
  }
  protected void doRegisterBeanDefinitions(Element root) {
		// Any nested <beans> elements will cause recursion in this method. In
		// order to propagate and preserve <beans> default-* attributes correctly,
		// keep track of the current (parent) delegate, which may be null. Create
		// the new (child) delegate with a reference to the parent for fallback purposes,
		// then ultimately reset this.delegate back to its original (parent) reference.
		// this behavior emulates a stack of delegates without actually necessitating one.

		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isInfoEnabled()) {
						logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}
//在解析Bean定义之前，进行自定义的解析，增强解析过程的可扩展性  
		preProcessXml(root);
//从Document的根元素开始进行Bean定义的Document对象  
		parseBeanDefinitions(root, this.delegate);
//在解析Bean定义之后，进行自定义的解析，增加解析过程的可扩展性      
		postProcessXml(root);

		this.delegate = parent;
	}
  //使用Spring的Bean规则从Document的根元素开始进行Bean定义的Document对象  
  protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
  		if (delegate.isDefaultNamespace(root)) {
  			NodeList nl = root.getChildNodes();
  			for (int i = 0; i < nl.getLength(); i++) {
  				Node node = nl.item(i);
  				if (node instanceof Element) {
  					Element ele = (Element) node;
  					if (delegate.isDefaultNamespace(ele)) {
  						parseDefaultElement(ele, delegate);
  					}
  					else {
  						delegate.parseCustomElement(ele);
  					}
  				}
  			}
  		}
  		else {
  			delegate.parseCustomElement(root);
  		}
  	}
//使用Spring的Bean规则解析Document元素节点  
  	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
  		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
  			importBeanDefinitionResource(ele);
  		}
  		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
  			processAliasRegistration(ele);
  		}
  		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
  			processBeanDefinition(ele, delegate);
  		}
  		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
  			// recurse
  			doRegisterBeanDefinitions(ele);
  		}
  	}

  	/**
  	 * Parse an "import" element and load the bean definitions
  	 * from the given resource into the bean factory.
  	 */
  	protected void importBeanDefinitionResource(Element ele) {
  		String location = ele.getAttribute(RESOURCE_ATTRIBUTE);
  		if (!StringUtils.hasText(location)) {
  			getReaderContext().error("Resource location must not be empty", ele);
  			return;
  		}

  		// Resolve system properties: e.g. "${user.dir}"
  		location = getReaderContext().getEnvironment().resolveRequiredPlaceholders(location);

  		Set<Resource> actualResources = new LinkedHashSet<Resource>(4);

  		// Discover whether the location is an absolute or relative URI
  		boolean absoluteLocation = false;
  		try {
  			absoluteLocation = ResourcePatternUtils.isUrl(location) || ResourceUtils.toURI(location).isAbsolute();
  		}
  		catch (URISyntaxException ex) {
  			// cannot convert to an URI, considering the location relative
  			// unless it is the well-known Spring prefix "classpath*:"
  		}

  		// Absolute or relative?
  		if (absoluteLocation) {
  			try {
  				int importCount = getReaderContext().getReader().loadBeanDefinitions(location, actualResources);
  				if (logger.isDebugEnabled()) {
  					logger.debug("Imported " + importCount + " bean definitions from URL location [" + location + "]");
  				}
  			}
  			catch (BeanDefinitionStoreException ex) {
  				getReaderContext().error(
  						"Failed to import bean definitions from URL location [" + location + "]", ele, ex);
  			}
  		}
  		else {
  			// No URL -> considering resource location as relative to the current file.
  			try {
  				int importCount;
  				Resource relativeResource = getReaderContext().getResource().createRelative(location);
  				if (relativeResource.exists()) {
  					importCount = getReaderContext().getReader().loadBeanDefinitions(relativeResource);
  					actualResources.add(relativeResource);
  				}
  				else {
  					String baseLocation = getReaderContext().getResource().getURL().toString();
  					importCount = getReaderContext().getReader().loadBeanDefinitions(
  							StringUtils.applyRelativePath(baseLocation, location), actualResources);
  				}
  				if (logger.isDebugEnabled()) {
  					logger.debug("Imported " + importCount + " bean definitions from relative location [" + location + "]");
  				}
  			}
  			catch (IOException ex) {
  				getReaderContext().error("Failed to resolve current resource location", ele, ex);
  			}
  			catch (BeanDefinitionStoreException ex) {
  				getReaderContext().error("Failed to import bean definitions from relative location [" + location + "]",
  						ele, ex);
  			}
  		}
  		Resource[] actResArray = actualResources.toArray(new Resource[actualResources.size()]);
  		getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
  	}

  	/**
  	 * Process the given alias element, registering the alias with the registry.
  	 */
  	protected void processAliasRegistration(Element ele) {
      ...
  	}

  	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
  		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
  		if (bdHolder != null) {
  			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
  			try {
  				// Register the final decorated instance.
  				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
  			}
  			catch (BeanDefinitionStoreException ex) {
  				getReaderContext().error("Failed to register bean definition with name '" +
  						bdHolder.getBeanName() + "'", ele, ex);
  			}
  			// Send registration event.
  			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
  		}
  	}

}
```

通过以上代码，可以看出：
通过上述Spring IoC容器对载入的Bean定义Document解析可以看出，我们使用Spring时，在Spring配置文件中可以使用<Import>元素来导入IoC容器所需要的其他资源，Spring IoC容器在解析时会首先将指定导入的资源加载进容器中。使用<Ailas>别名时，Spring IoC容器首先将别名元素所定义的别名注册到容器中。
对于既不是<Import>元素，又不是<Alias>元素的元素，即Spring配置文件中普通的<Bean>元素的解析由BeanDefinitionParserDelegate类的parseBeanDefinitionElement方法来实现。

##### 3.2.4.5 BeanDefinitionParserDelegate解析Bean定义资源文件中的<Bean>元素

```Java
/**
 * Parses the supplied {@code <bean>} element. May return {@code null}
 * if there were errors during parse. Errors are reported to the
 * {@link org.springframework.beans.factory.parsing.ProblemReporter}.
 */
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
  return parseBeanDefinitionElement(ele, null);
}
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
  String id = ele.getAttribute(ID_ATTRIBUTE);
  String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

  List<String> aliases = new ArrayList<String>();
  if (StringUtils.hasLength(nameAttr)) {
    String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
    aliases.addAll(Arrays.asList(nameArr));
  }

  String beanName = id;
  if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
    beanName = aliases.remove(0);
    if (logger.isDebugEnabled()) {
      logger.debug("No XML 'id' specified - using '" + beanName +
          "' as bean name and " + aliases + " as aliases");
    }
  }

  if (containingBean == null) {
    checkNameUniqueness(beanName, aliases, ele);
  }

  AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
  if (beanDefinition != null) {
    if (!StringUtils.hasText(beanName)) {
      try {
        if (containingBean != null) {
          beanName = BeanDefinitionReaderUtils.generateBeanName(
              beanDefinition, this.readerContext.getRegistry(), true);
        }
        else {
          beanName = this.readerContext.generateBeanName(beanDefinition);
          // Register an alias for the plain bean class name, if still possible,
          // if the generator returned the class name plus a suffix.
          // This is expected for Spring 1.2/2.0 backwards compatibility.
          String beanClassName = beanDefinition.getBeanClassName();
          if (beanClassName != null &&
              beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
              !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
            aliases.add(beanClassName);
          }
        }
        if (logger.isDebugEnabled()) {
          logger.debug("Neither XML 'id' nor 'name' specified - " +
              "using generated bean name [" + beanName + "]");
        }
      }
      catch (Exception ex) {
        error(ex.getMessage(), ele);
        return null;
      }
    }
    String[] aliasesArray = StringUtils.toStringArray(aliases);
    return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
  }

  return null;
}
public AbstractBeanDefinition parseBeanDefinitionElement(
    Element ele, String beanName, BeanDefinition containingBean) {

  this.parseState.push(new BeanEntry(beanName));

  String className = null;
  if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
    className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
  }

  try {
    String parent = null;
    if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
      parent = ele.getAttribute(PARENT_ATTRIBUTE);
    }
    //根据<Bean>元素配置的class名称和parent属性值创建BeanDefinition  
    //为载入Bean定义信息做准备  
    AbstractBeanDefinition bd = createBeanDefinition(className, parent);  
    //对当前的<Bean>元素中配置的一些属性进行解析和设置，如配置的单态(singleton)属性等  
    parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);  
    //为<Bean>元素解析的Bean设置description信息
    bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));  

    //对<Bean>元素的meta(元信息)属性解析  
    parseMetaElements(ele, bd);  
    //对<Bean>元素的lookup-method属性解析  
    parseLookupOverrideSubElements(ele, bd.getMethodOverrides());  
    //对<Bean>元素的replaced-method属性解析  
    parseReplacedMethodSubElements(ele, bd.getMethodOverrides());  
    //解析<Bean>元素的构造方法设置  

    parseConstructorArgElements(ele, bd);  
    //解析<Bean>元素的<property>设置  
    parsePropertyElements(ele, bd);  
    //解析<Bean>元素的qualifier属性  
    parseQualifierElements(ele, bd);  
    //为当前解析的Bean设置所需的资源和依赖对象  
    bd.setResource(this.readerContext.getResource());  
    bd.setSource(extractSource(ele));

    return bd;
  }
  catch (ClassNotFoundException ex) {
    error("Bean class [" + className + "] not found", ele, ex);
  }
  catch (NoClassDefFoundError err) {
    error("Class that bean class [" + className + "] depends on not found", ele, err);
  }
  catch (Throwable ex) {
    error("Unexpected failure during bean definition parsing", ele, ex);
  }
  finally {
    this.parseState.pop();
  }

  return null;
}

```
只要使用过Spring，对Spring配置文件比较熟悉的人，通过对上述源码的分析，就会明白我们在Spring配置文件中<Bean>元素的中配置的属性就是通过该方法解析和设置到Bean中去的。

注意：在解析<Bean>元素过程中没有创建和实例化Bean对象，只是创建了Bean对象的定义类BeanDefinition，将<Bean>元素中的配置信息设置到BeanDefinition中作为记录，当依赖注入时才使用这些记录信息创建和实例化具体的Bean对象。



#### 3.2.5 DefaultListableBeanFactory向IoC容器注册解析后的BeanDefinition
```Java
public class DefaultBeanDefinitionDocumentReader implements BeanDefinitionDocumentReader {
    protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
      BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
      if (bdHolder != null) {
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
          // Register the final decorated instance.
          BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {
          getReaderContext().error("Failed to register bean definition with name '" +
              bdHolder.getBeanName() + "'", ele, ex);
        }
        // Send registration event.
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
      }
    }
}
```
在这里 DefaultBeanDefinitionDocumentReader调用了BeanDefinitionReaderUtils.registerBeanDefinition
```Java
public static void registerBeanDefinition(
    BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
    throws BeanDefinitionStoreException {

  // Register bean definition under primary name.
  String beanName = definitionHolder.getBeanName();
  //向IoC容器注册BeanDefinition
  registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

  // Register aliases for bean name, if any.
  String[] aliases = definitionHolder.getAliases();
  if (aliases != null) {
    for (String alias : aliases) {
      registry.registerAlias(beanName, alias);
    }
  }
}
```

##### 3.2.5.1 DefaultListableBeanFactory向IoC容器注册解析后的BeanDefinition：
DefaultListableBeanFactory中使用一个HashMap的集合对象存放IoC容器中注册解析的BeanDefinition，向IoC容器注册的主要源码如下：
```Java
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
		implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {
      /** Map of bean definition objects, keyed by bean name */
      private final Map<String, BeanDefinition> beanDefinitionMap =
          new ConcurrentHashMap<String, BeanDefinition>(256);

      public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
      			throws BeanDefinitionStoreException {

      		Assert.hasText(beanName, "Bean name must not be empty");
      		Assert.notNull(beanDefinition, "BeanDefinition must not be null");

      		if (beanDefinition instanceof AbstractBeanDefinition) {
      			try {
      				((AbstractBeanDefinition) beanDefinition).validate();
      			}
      			catch (BeanDefinitionValidationException ex) {
      				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
      						"Validation of bean definition failed", ex);
      			}
      		}

      		BeanDefinition oldBeanDefinition;

      		oldBeanDefinition = this.beanDefinitionMap.get(beanName);
      		if (oldBeanDefinition != null) {
      			if (!isAllowBeanDefinitionOverriding()) {
      				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
      						"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
      						"': There is already [" + oldBeanDefinition + "] bound.");
      			}
      			else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
      				// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
      				if (this.logger.isWarnEnabled()) {
      					this.logger.warn("Overriding user-defined bean definition for bean '" + beanName +
      							"' with a framework-generated bean definition: replacing [" +
      							oldBeanDefinition + "] with [" + beanDefinition + "]");
      				}
      			}
      			else if (!beanDefinition.equals(oldBeanDefinition)) {
      				if (this.logger.isInfoEnabled()) {
      					this.logger.info("Overriding bean definition for bean '" + beanName +
      							"' with a different definition: replacing [" + oldBeanDefinition +
      							"] with [" + beanDefinition + "]");
      				}
      			}
      			else {
      				if (this.logger.isDebugEnabled()) {
      					this.logger.debug("Overriding bean definition for bean '" + beanName +
      							"' with an equivalent definition: replacing [" + oldBeanDefinition +
      							"] with [" + beanDefinition + "]");
      				}
      			}
      			this.beanDefinitionMap.put(beanName, beanDefinition);
      		}
      		else {
      			if (hasBeanCreationStarted()) {
      				// Cannot modify startup-time collection elements anymore (for stable iteration)
              //注册的过程中需要线程同步，以保证数据的一致性
      				synchronized (this.beanDefinitionMap) {
      					this.beanDefinitionMap.put(beanName, beanDefinition);
      					List<String> updatedDefinitions = new ArrayList<String>(this.beanDefinitionNames.size() + 1);
      					updatedDefinitions.addAll(this.beanDefinitionNames);
      					updatedDefinitions.add(beanName);
      					this.beanDefinitionNames = updatedDefinitions;
      					if (this.manualSingletonNames.contains(beanName)) {
      						Set<String> updatedSingletons = new LinkedHashSet<String>(this.manualSingletonNames);
      						updatedSingletons.remove(beanName);
      						this.manualSingletonNames = updatedSingletons;
      					}
      				}
      			}
      			else {
      				// Still in startup registration phase
      				this.beanDefinitionMap.put(beanName, beanDefinition);
      				this.beanDefinitionNames.add(beanName);
      				this.manualSingletonNames.remove(beanName);
      			}
      			this.frozenBeanDefinitionNames = null;
      		}

      		if (oldBeanDefinition != null || containsSingleton(beanName)) {
      			resetBeanDefinition(beanName);
      		}
      }
}

```

至此，Bean定义资源文件中配置的Bean被解析过后，已经注册到IoC容器中，被容器管理起来，真正完成了IoC容器初始化所做的全部工作。现  在IoC容器中已经建立了整个Bean的配置信息，这些BeanDefinition信息已经可以使用，并且可以被检索，IoC容器的作用就是对这些注册的Bean定义信息进行处理和维护。这些的注册的Bean定义信息是IoC容器控制反转的基础，正是有了这些注册的数据，容器才可以进行依赖注入。

### 3.3 总结IOC容器的基本步骤

1. 初始化的入口在容器实现中的 refresh()调用来完成.
2. 对 bean 定义载入 IOC 容器使用的方法是 loadBeanDefinition,其中的大致过程如下：
- 为容器设置Bean资源加载器。AbstractApplicationContext构造方法中调用PathMatchingResourcePatternResolver的构造方法创建Spring资源加载器。
- 设置Bean定义资源文件的定位路径。
- refresh函数载入Bean定义过程。
  - obtainFreshBeanFactory()方法 刷新容器.
  - BeanDefinition 读取Bean定义资源。读取资源转换为Document。DocumentLoader解析xml文件。XmlBeanDefinitionReader解析载入的Bean定义资源文件。BeanDefinitionParserDelegate解析Bean定义资源文件中的<Bean>元素。DefaultListableBeanFactory向IoC容器注册解析后的BeanDefinition。DefaultListableBeanFactory向IoC容器注册解析后的BeanDefinition。

- 通过 ResourceLoader 来完成资源文件位置的定位，DefaultResourceLoader 是默认的实现，同时上下文本身就给出了 ResourceLoader 的实现，可以从类路径，文件系统, URL 等方式来定为资源位置。如果是 XmlBeanFactory作为 IOC 容器，那么需要为它指定 bean 定义的资源，也就是说 bean 定义文件时通过抽象成 Resource 来被 IOC 容器处理的，容器通过 BeanDefinitionReader来完成定义信息的解析和 Bean 信息的注册,往往使用的是XmlBeanDefinitionReader 来解析 bean 的 xml 定义文件 - 实际的处理过程是委托给 BeanDefinitionParserDelegate 来完成的，从而得到 bean 的定义信息，这些信息在 Spring 中使用 BeanDefinition 对象来表示 - 这个名字可以让我们想到loadBeanDefinition,RegisterBeanDefinition  这些相关的方法 - 他们都是为处理 BeanDefinitin 服务的， 容器解析得到 BeanDefinitionIoC 以后，需要把它在 IOC 容器中注册，这由 IOC 实现 BeanDefinitionRegistry 接口来实现。注册过程就是在 IOC 容器内部维护的一个HashMap 来保存得到的 BeanDefinition 的过程。这个 HashMap 是 IoC 容器持有 bean 信息的场所，以后对 bean 的操作都是围绕这个HashMap 来实现的。

3. 然后我们就可以通过 BeanFactory 和 ApplicationContext 来享受到 Spring IOC 的服务了,在使用 IOC 容器的时候，我们注意到除了少量粘合代码，绝大多数以正确 IoC 风格编写的应用程序代码完全不用关心如何到达工厂，因为容器将把这些对象与容器管理的其他对象钩在一起。基本的策略是把工厂放到已知的地方，最好是放在对预期使用的上下文有意义的地方，以及代码将实际需要访问工厂的地方。 Spring 本身提供了对声明式载入 web 应用程序用法的应用程序上下文,并将其存储在ServletContext 中的框架实现。

>  Beanfactory 和 Factory bean，其中 BeanFactory 指的是 IOC 容器的编程抽象，比如 ApplicationContext， XmlBeanFactory 等，这些都是 IOC 容器的具体表现，需要使用什么样的容器由客户决定,但 Spring 为我们提供了丰富的选择。 FactoryBean 只是一个可以在 IOC而容器中被管理的一个 bean,是对各种处理过程和资源使用的抽象,Factory bean 在需要时产生另一个对象，而不返回 FactoryBean本身,我们可以把它看成是一个抽象工厂，对它的调用返回的是工厂生产的产品。所有的 Factory bean 都实现特殊的org.springframework.beans.factory.FactoryBean 接口，当使用容器中 factory bean 的时候，该容器不会返回 factory bean 本身,而是返回其生成的对象。Spring 包括了大部分的通用资源和服务访问抽象的 Factory bean 的实现，其中包括:对 JNDI 查询的处理，对代理对象的处理，对事务性代理的处理，对 RMI 代理的处理等，这些我们都可以看成是具体的工厂,看成是SPRING 为我们建立好的工厂。也就是说 Spring 通过使用抽象工厂模式为我们准备了一系列工厂来生产一些特定的对象,免除我们手工重复的工作，我们要使用时只需要在 IOC 容器里配置好就能很方便的使用了。

## IOC容器的依赖注入
















































[1]:https://images0.cnblogs.com/blog/400827/201409/172219470349285.x-png
[2]:https://s26.postimg.org/5pisndljd/spring_beandefinition.png
[3]:https://s26.postimg.org/s2qjah4k9/classpathxmlapplicaitoncontext.png
[4]:https://s26.postimg.org/vmch0e9vt/xmlbean_Factory.png
[5]:https://s26.postimg.org/d0s358tdl/Abstract_Bean_Definition_Reader.png
