# `Interceptor`的使用方法

>本文前三章节全文copy自作者`维C果糖`,文章出自`CSDN`。本文仅做学习记录使用。
>如果想更详细的了解文章，请[点击][1]去原文。表达对博主的感谢。

> 代码被提交到[github][4]

## 1. 拦截器概念
`Java`里的拦截器是动态拦截`action`调用的对象。它提供了一种机制可以使开发者可以定义在一个`action`执行的前后执行的代码，也可以在一个`action`执行前阻止其执行，同时也提供了一种可以提取`action`中可重用部分的方式。在`AOP`（`Aspect-OrientedProgramming`，面向切面编程）中拦截器用于在某个方法或字段被访问之前进行拦截，然后在之前或之后加入某些操作。


## 2. 实现方法

在`Spring`框架之中，咱们要想实现拦截器的功能，主要通过两种途径，第一种是实现`HandlerInterceptor`接口，第二种是实现`WebRequestInterceptor`接口。接下来，咱们分别详细的介绍两者的实现方法。

### 2.1 `HandlerInterceptor`接口
在`HandlerInterceptor`接口中，定义了3个方法，分别为`preHandle()`、`postHandle()`和`afterCompletion()`，咱们就是通过复写这3个方法来对用户的请求进行拦截处理的。因此，咱们可以通过直接实现`HandlerInterceptor`接口来实现拦截器的功能。不过在`Spring`框架之中，其还提供了另外一个接口和一个抽象类，实现了对`HandlerInterceptor`接口的功能扩展，分别为：`AsyncHandlerInterceptor`和`HandlerInterceptorAdapter`.

对于`AsyncHandlerInterceptor`接口，其在继承`HandlerInterceptor`接口的同时，又声明了一个新的方法a`fterConcurrentHandlingStarted()`；而`HandlerInterceptorAdapter`抽象类，则是更进一步，在其继承`AsyncHandlerInterceptor`接口的同时，又复写了`preHandle`方法。因此，`AsyncHandlerInterceptor`更像是一个过渡的接口。

在实际应用中，咱们一般都是通过实现`HandlerInterceptor`接口或者继承`HandlerInterceptorAdapter`抽象类，复写`preHandle()`、`postHandle()`和`afterCompletion()`这3个方法来对用户的请求进行拦截处理的。下面，咱们就详细介绍这个3个方法。


- `preHandle(HttpServletRequestrequest,HttpServletResponseresponse,Objecthandle)`方法，该方法在请求处理之前进行调用。`SpringMVC`中的`Interceptor`是链式调用的，在一个应用中或者说是在一个请求中可以同时存在多个`Interceptor`。每个`Interceptor`的调用会依据它的声明顺序依次执行，而且最先执行的都是`Interceptor`中的`preHandle`方法，所以可以在这个方法中**进行一些前置初始化操作或者是对当前请求做一个预处理**，也可以在这个方法中**进行一些判断来决定请求是否要继续进行下去**。该方法的返回值是布尔值`Boolean`类型的，当它返回为`false`时，表示请求结束，后续的`Interceptor`和`Controller`都不会再执行；当返回值为`true`时，就会继续调用下一个`Interceptor`的`preHandle`方法，如果已经是最后一个`Interceptor`的时候，就会是调用当前请求的`Controller`中的方法。

- `postHandle(HttpServletRequestrequest,HttpServletResponseresponse,Objecthandle,ModelAndViewmodelAndView)`方法，通过`preHandle`方法的解释咱们知道这个方法包括后面要说到的`afterCompletion`方法都只能在当前所属的`Interceptor`的`preHandle`方法的返回值为`true`的时候，才能被调用。`postHandle`方法在当前请求进行处理之后，也就是在`Controller`中的方法调用之后执行，但是它会在`DispatcherServlet`进行视图返回渲染之前被调用，所以咱们可以在这个方法中对`Controller`处理之后的`ModelAndView`对象进行操作。`postHandle`方法被调用的方向跟`preHandle`是相反的，也就是说，先声明的`Interceptor`的`postHandle`方法反而会后执行。这和`Struts2`里面的`Interceptor`的执行过程有点类型，`Struts2`里面的`Interceptor`的执行过程也是链式的，只是在`Struts2`里面需要手动调用`ActionInvocation`的`invoke`方法来触发对下一个`Interceptor`或者是`action`的调用，然后每一个`Interceptor`中在`invoke`方法调用之前的内容都是按照声明顺序执行的，而`invoke`方法之后的内容就是反向的。

- `afterCompletion(HttpServletRequestrequest,HttpServletResponseresponse,Objecthandle,Exceptionex)`方法，也是需要当前对应的`Interceptor`的`preHandle`方法的返回值为`true`时才会执行。因此，该方法将在整个请求结束之后，也就是在`DispatcherServlet`渲染了对应的视图之后执行，这个方法的主要作用是用于进行资源清理的工作。


**`HandlerInterceptor`接口:**
```Java
public interface HandlerInterceptor {

	boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception;

	void postHandle(
			HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)
			throws Exception;

	void afterCompletion(
			HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
			throws Exception;
}
```

**`AsyncHandlerInterceptor`接口:**
```Java
public interface AsyncHandlerInterceptor extends HandlerInterceptor {


	void afterConcurrentHandlingStarted(HttpServletRequest request,
		HttpServletResponse response, Object handler)
			throws Exception;

}

```

**`HandlerInterceptorAdapter` 抽象类：**

```Java
public abstract class HandlerInterceptorAdapter implements AsyncHandlerInterceptor {

	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		return true;
	}

	@Override
	public void postHandle(
			HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)
			throws Exception {
	}

	@Override
	public void afterCompletion(
			HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
			throws Exception {
	}

	@Override
	public void afterConcurrentHandlingStarted(
			HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
	}

}

```

如上面的代码所示，其实在`HandlerInterceptor`和`AsyncHandlerInterceptor`中还有很多的代码注释，只是感觉太多了，就将其全部删除啦！如果大家对这些注释感兴趣的话，可以自行查看源代码。下面，咱们以继承`HandlerInterceptorAdapter`抽象类为例进行演示：

```Java
@Component
public class WebHandlerInterceptor extends HandlerInterceptorAdapter {
    @Autowired
    ThreadLocal<Date> threadLocal;

    private String name = "cache-name";


    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("preHandler");
        response.setHeader(name,"");
        return super.preHandle(request,response,handler);
    }


    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
			//注意，这里的setHeader不会生效，下文会有介绍。
        String value = threadLocal.get().toString();
        response.setHeader(name, threadLocal.get().toString());
        super.postHandle(request,response,handler,modelAndView);
    }

}
```

### 2.2 `WebRequestInterceptor`接口
在`WebRequestInterceptor`接口中也定义了3个方法，同`HandlerInterceptor`接口完全相同，咱们也是通过复写这3个方法来用户的请求进行拦截处理的。而且这3个方法都传递了同一个参数`WebRequest`，那么这个`WebRequest`到底是什么呢？其实这个`WebRequest`是`Spring`中定义的一个接口，它里面的方法定义跟`HttpServletRequest`类似，在`WebRequestInterceptor`中对`WebRequest`进行的所有操作都将同步到`HttpServletRequest`中，然后在当前请求中依次传递。

在`Spring`框架之中，还提供了一个和`WebRequestInterceptor`接口长的很像的抽象类，那就是：`WebRequestInterceptorAdapter`，其实现了`AsyncHandlerInterceptor`接口，并在内部调用了`WebRequestInterceptor`接口。

接下来，咱们主要讲一下`WebRequestInterceptor`接口的3个函数

- `preHandle(WebRequestrequest)`方法，该方法在请求处理之前进行调用，也就是说，其会在`Controller`中的方法调用之前被调用。这个方法跟`HandlerInterceptor`中的`preHandle`不同，主要区别在于该方法的返回值是`void`类型的，也就是没有返回值，因此我们主要用它来进行资源的准备工作，比如我们在使用`Hibernate`的时候，可以在这个方法中准备一个`Hibernate`的`Session`对象，然后利用`WebRequest`的`setAttribute(name,value,scope)`把它放到`WebRequest`的属性中。在这里，进一步说说`setAttribute`方法的第三个参数`scope`，该参数是一个`Integer`类型的。在`WebRequest`的父层接口`RequestAttributes`中对它定义了三个常量，分别为：
	+ `SCOPE_REQUEST`，它的值是0，表示只有在`request`中可以访问。
	+ `SCOPE_SESSION`，它的值是1，如果环境允许的话，它表示的是一个局部的隔离的`session`，否则就代表普通的`session`，并且在该`session`范围内可以访问。
	+ `SCOPE_GLOBAL_SESSION`，它的值是2，如果环境允许的话，它表示的是一个全局共享的`session`，否则就代表普通的`session`，并且在该`session`范围内可以访问。

- `postHandle(WebRequestrequest,ModelMapmodel)`方法，该方法在请求处理之后，也就是在`Controller`中的方法调用之后被调用，但是会在视图返回被渲染之前被调用，所以可以在这个方法里面通过改变数据模型`ModelMap`来改变数据的展示。该方法有两个参数，`WebRequest`对象是用于传递整个请求数据的，比如在`preHandle`中准备的数据都可以通过`WebRequest`来传递和访问；`ModelMap`就是`Controller`处理之后返回的`Model`对象，咱们可以通过改变它的属性来改变返回的`Model`模型。

- `afterCompletion(WebRequestrequest,Exceptionex)`方法，该方法会在整个请求处理完成，也就是在视图返回并被渲染之后执行。因此可以在该方法中进行资源的释放操作。而`WebRequest`参数就可以把咱们在`preHandle`中准备的资源传递到这里进行释放。`Exception`参数表示的是当前请求的异常对象，**如果在Controller中抛出的异常已经被Spring的异常处理器给处理了的话，那么这个异常对象就是是null**.

**`WebRequestInterceptor`接口:**

```Java
public interface WebRequestInterceptor {

	void preHandle(WebRequest request) throws Exception;

	void postHandle(WebRequest request, ModelMap model) throws Exception;

	void afterCompletion(WebRequest request, Exception ex) throws Exception;

}

```

**`WebRequestHandlerInterceptorAdapter`接口:**
```java
public class WebRequestHandlerInterceptorAdapter implements AsyncHandlerInterceptor {

	private final WebRequestInterceptor requestInterceptor;

	public WebRequestHandlerInterceptorAdapter(WebRequestInterceptor requestInterceptor) {
		Assert.notNull(requestInterceptor, "WebRequestInterceptor must not be null");
		this.requestInterceptor = requestInterceptor;
	}


	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		this.requestInterceptor.preHandle(new DispatcherServletWebRequest(request, response));
		return true;
	}

	@Override
	public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)
			throws Exception {

		this.requestInterceptor.postHandle(new DispatcherServletWebRequest(request, response),
				(modelAndView != null && !modelAndView.wasCleared() ? modelAndView.getModelMap() : null));
	}

	@Override
	public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
			throws Exception {

		this.requestInterceptor.afterCompletion(new DispatcherServletWebRequest(request, response), ex);
	}

	@Override
	public void afterConcurrentHandlingStarted(HttpServletRequest request, HttpServletResponse response, Object handler) {
		if (this.requestInterceptor instanceof AsyncWebRequestInterceptor) {
			AsyncWebRequestInterceptor asyncInterceptor = (AsyncWebRequestInterceptor) this.requestInterceptor;
			DispatcherServletWebRequest webRequest = new DispatcherServletWebRequest(request, response);
			asyncInterceptor.afterConcurrentHandlingStarted(webRequest);
		}
	}

}

```

### 2.3 `AbstractInterceptor`抽象类

除了上面`2.1`和`2.3`所讲的内容，咱们还可以通过继承`Struts2`框架提供的`AbstractInterceptor`抽象类来实现拦截的功能。如果咱们在深入一点研究，会发现`AbstractInterceptor`实现了`Interceptor`接口，而`Interceptor`接口又继承了`Serializable`接口。

在`Interceptor`接口中，提供了3个方法供咱们使用，分别为`init()`、`destroy()`和`intercept()`，由于`AbstractInterceptor`实现了`Interceptor`接口，因此咱们就可以直接继承`AbstractInterceptor`，然后复写方法就可以啦！至于为什么继承`AbstractInterceptor`而不是直接实现`Interceptor`接口，是因为`AbstractInterceptor`已经帮咱们实现了空的`init()`和`destroy()`方法，不需要咱们自己去复写了，咱们直接复写`intercept()`方法就可以啦！现在，咱们大致了解一下这3个方法的作用：

- `init()`方法，一般用来进行初始化操作；
`destroy()`方法，一般用来进行释放资源的操作；
`intercept()`方法，该方法是实现拦截功能的主要方法，咱们就在该方法中编写拦截的逻辑。
```Java
public interface Interceptor extends Serializable {

    /**
     * Called to let an interceptor clean up any resources it has allocated.
     */
    void destroy();

    /**
     * Called after an interceptor is created, but before any requests are processed using
     * {@link #intercept(com.opensymphony.xwork2.ActionInvocation) intercept} , giving
     * the Interceptor a chance to initialize any needed resources.
     */
    void init();

    /**
     * Allows the Interceptor to do some processing on the request before and/or after the rest of the processing of the
     * request by the {@link ActionInvocation} or to short-circuit the processing and just return a String return code.
     *
     * @param invocation the action invocation
     * @return the return code, either returned from {@link ActionInvocation#invoke()}, or from the interceptor itself.
     * @throws Exception any system-level error, as defined in {@link com.opensymphony.xwork2.Action#execute()}.
     */
    String intercept(ActionInvocation invocation) throws Exception;

}
```

**`AbstractInterceptor`接口：**
```Java
public abstract class AbstractInterceptor implements Interceptor {

    public void init() {
    }

    public void destroy() {
    }

    public abstract String intercept(ActionInvocation invocation) throws Exception;
}
```

## 3. 配置拦截器

在前面，咱们用了很大篇幅的内容讲述了拦截器如何实现，因此，我相信大家实现拦截器已经没有问题啦！接下来，咱们在看看，如何在`XML`文件中配置拦截器，使咱们的拦截器生效。

在配置拦截器之前，有4个名称的概念需要大家先了解一下，分别为：`JoinPoint`、`Pointcut`、`Advice`和`Advisor`.

`JoinPoint`，表示“连接点”，它是程序运行中的某个阶段点，比如方法的调用、异常的抛出等；
`Advice`，表示“通知”，它是某个连接点所采用的处理逻辑，也就是向连接点注入的代码；
`Pointcut`，表示“切入点”，它是“连接点”的集合，是程序中需要注入`Advice`的位置的集合，指明`Advice`要在什么样的条件下才能被触发；
`Advisor`，它是`Pointcut`和`Advice`的配置器，包括`Pointcut`和`Advice`，是将`Advice`注入程序中`Pointcut`位置的代码。

### 3.1 基于`XML`的配置
而在`mvc`标签中有一个名为`mvc:interceptors`的标签，该标签就是用于声明`Spring`拦截器的。下面，给出一个配置示例：

```xml
<mvc:interceptors>  
    <!-- 使用 bean 定义一个 Interceptor，直接定义在 mvc:interceptors 下面的 Interceptor 将拦截所有的请求 -->  
    <bean class="com.hit.interceptor.WrongCodeInterceptor"/>  
    <mvc:interceptor>  
        <mvc:mapping path="/demo/hello.do"/>  
        <!-- 定义在 mvc:interceptor 下面的 Interceptor，表示对特定的请求进行拦截 -->  
        <bean class="com.hit.interceptor.LoginInterceptor"/>  
    </mvc:interceptor>  
</mvc:interceptors>  
```
在 Spring 的XML 配置文件中，咱们可以通过mvc:interceptors标签声明一系列的拦截器，例如：
```xml
<mvc:interceptors>
        <bean class="com.hit.interceptor.ContextInterceptor"/>
        <bean class="com.hit.interceptor.LoginInterceptor"/>
        <bean class="com.hit.interceptor.WrongCodeInterceptor"/>
    </mvc:interceptors>
```


如上所示，这些拦截器就够成了一个拦截器链，或者称之为拦截器栈。而这些拦截器的执行顺序是按声明的先后顺序执行的，即：先声明的拦截器先执行，后声明的拦截器后执行。在`mvc:interceptors`标签下声明`interceptor`标签主要有两种方式：

直接定义一个`Interceptor`实现类的`bean`对象，使用这种方式声明的`Interceptor`拦截器将会对所有的请求进行拦截；
使用`mvc:interceptor`标签进行声明，使用这种方式进行声明的`Interceptor`可以通过`mvc:mapping`子标签来定义需要进行拦截的请求路径。
此外，由于拦截器是`AOP`编程思想的典型应用，也就意味着咱们可以“切”到具体的“面”进行某些操作。例如，

```xml
<bean id="WrongCodeInterceptor" class="com.hit.interceptor.WrongCodeInterceptor">
        <property name="userName" value="user-module"></property>
</bean>

<bean id="loginInterceptor" class="com.hit.interceptor.LoginInterceptor">
    <property name="excludePackages">
       <list>
          <value>com.hit.user.exception</value>
          <value>com.hit.order.exception</value>
       </list>
    </property>
</bean>

<aop:config>
    <aop:advisor advice-ref="WrongCodeInterceptor" pointcut="execution(* com.hit.*.demo..*.*(..)) || execution(* com.hit.*.demo..*.*(..)) " />
    <aop:advisor advice-ref="loginInterceptor" pointcut="execution(* com.hit.*.demo..*.*(..))" />
</aop:config>
```
如上所示，咱们实现了切入到“面”进行特定的拦截功能，其中`pointcut`表示“切入点”，`advisor`表示要注入到`pointcut`的代码。大家可能会对`pointcut`中的*符合有所疑惑，它是“通配符”，表示可以匹配该位置上的任何名称。当然，如果咱们要想使用`aop`标签，就得先在配置文件中就得进行声明啦！

### 3.2 基于`Java configuration`的配置

在Spring中, 通过继承`WebMvcConfigurerAdapter`可以实现上面的效果。

但是请注意，`Interceptor`的载入顺序在`Bean`之前，所以，如果想要在`interceptor`中注入`Bean`的话，要在`WebMvcConfigurerAdapter`中先声明一个`Interceptor Bean`，然后引入`Interceptor`。
```Java
@Configuration
public class WebMvcConfigureRegistration extends WebMvcConfigurerAdapter {

    @Bean
    public WebHandlerInterceptor webHandlerInterceptor() {
        return new WebHandlerInterceptor();
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(webHandlerInterceptor());
        super.addInterceptors(registry);
    }
}

```

## 4. 注意事项
### 4.1. 如何在`controller`执行之后，修改`Response Header`.
做了这么个项目：
数据返回给前台的时候，大部分数据是从缓存里拿到的数据，所以要把缓存的时间返回给客户端，业务问题导致，只有当`Controller`执行之后，才能拿到`cache-time`。

#### 4.1.1 `HandlerInterceptorAdapter` 失败
所以最初的时候，尝试的是`HandlerInterceptorAdapter`去做的全局添加`Response Header`：

```Java
@Component
public class WebHandlerInterceptor extends HandlerInterceptorAdapter {
    @Autowired
    ThreadLocal<Date> threadLocal;

    private String name = "cache-name";

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("preHandler");
        response.setHeader(name,"");
        return super.preHandle(request,response,handler);
    }


    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
			//注意，这里的setHeader不会生效，下文会有介绍。
        String value = threadLocal.get().toString();
        response.setHeader(name, threadLocal.get().toString());
        super.postHandle(request,response,handler,modelAndView);
    }

}
```
但是结果，返回的`Response Header`是这样的：

```
cache-name:
```

也就是说，`preHandle`设置成功，但是`postHandle`并没有生效。

我们可以看到官方文档里面的说明：
>Note that the postHandle method of HandlerInterceptor is not always ideally suited for use with @ResponseBody and ResponseEntity methods. In such cases an HttpMessageConverter writes to and commits the response before postHandle is called which makes it impossible to change the response, for example to add a header. Instead an application can implement ResponseBodyAdvice and either declare it as an @ControllerAdvice bean or configure it directly on RequestMappingHandlerAdapter.

#### 4.1.2 `Filter` 失败
与`WebMvcConfigurerAdapter`相似。
在这里不再赘述。

#### 4.1.3 `\@ModelAttribute` 失败
**首先我们说说`@ModelAttribute`的作用：** 首先`@ModelAttribute`是使用在方法或参数上的
- 当使用在方法上时其作用于本身所在的`Controller`，在访问`Controller`中的所有请求时都会执行到`@ModelAttribute`所注解的方法。
- 作用于参数上的话...

`@ModelAttribute`一个具有如下三个作用：
1. 绑定请求参数到命令对象：放在功能处理方法的入参上时，用于将多个请求参数绑定到一个命令对象，从而简化绑定流程，而且自动暴露为模型数据用于视图页面展示时使用；
2. 暴露表单引用对象为模型数据：放在处理器的一般方法（非功能处理方法）上时，是为表单准备要展示的表单引用对象，如注册时需要选择的所在城市等，而且在执行功能处理方法（`@RequestMapping`注解的方法）之前，自动添加到模型对象中，用于视图页面展示时使用；
3. 暴露`@RequestMapping`方法返回值为模型数据：放在功能处理方法的返回值上时，是暴露功能处理方法的返回值为模型数据，用于视图页面展示时使用。


```Java
@RestController
public class TestController {

    @Autowired
    TestService service;

    @GetMapping(path = "/test", produces = "application/json")
    public String test() {
        return service.test();
    }

    @ModelAttribute
    public void setHeader(HttpServletResponse response) {
        System.out.println("ModelAttribute");
        response.setHeader("header", "cool");
    }
}

```
为什么`@ModelAttribute`注解的方法是作用于整个`Controller`的，实际上是在执行`Controller`的每个请求时都会执行`@ModelAttribute`注解的方法。

执行过程在`RequestMappingHandlerAdapter`中，每次执行`Controller`时会执行`@ModelAttribute`注解的方法：

```Java

protected ModelAndView invokeHandlerMethod(HttpServletRequest request, HttpServletResponse response,
 HandlerMethod handlerMethod) throws Exception {
    ......
    //执行@ModelAttribute注解的方法
    modelFactory.initModel(webRequest, mavContainer, invocableMethod);

    ......
    //执行Controller中的方法
    invocableMethod.invokeAndHandle(webRequest, mavContainer);
    ......

}
```

也就是说，**`@ModelAttribute`是在`controller`的`init`操作中执行的，这时候，`controller`里面的方法还没有执行**。

#### 4.1.4 `ResponseBodyAdvice`成功
先看看类的注释：
>Allows customizing the response after the execution of an `@ResponseBody` or a `ResponseEntity` controller method but before the body is written with an `HttpMessageConverter`.
Implementations may be registered directly with `RequestMappingHandlerAdapter` and `ExceptionHandlerExceptionResolver` or more likely annotated with `@ControllerAdvice` in which case they will be auto-detected by both.


**ResponseBodyAdvice<T>** 接口：
```Java
/**
 * Allows customizing the response after the execution of an {@code @ResponseBody}
 * or a {@code ResponseEntity} controller method but before the body is written
 * with an {@code HttpMessageConverter}.
 *
 * <p>Implementations may be registered directly with
 * {@code RequestMappingHandlerAdapter} and {@code ExceptionHandlerExceptionResolver}
 * or more likely annotated with {@code @ControllerAdvice} in which case they
 * will be auto-detected by both.
 *
 * @author Rossen Stoyanchev
 * @since 4.1
 */
public interface ResponseBodyAdvice<T> {

	/**
	 * Whether this component supports the given controller method return type
	 * and the selected {@code HttpMessageConverter} type.
	 * @param returnType the return type
	 * @param converterType the selected converter type
	 * @return {@code true} if {@link #beforeBodyWrite} should be invoked;
	 * {@code false} otherwise
	 */
	boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType);

	/**
	 * Invoked after an {@code HttpMessageConverter} is selected and just before
	 * its write method is invoked.
	 * @param body the body to be written
	 * @param returnType the return type of the controller method
	 * @param selectedContentType the content type selected through content negotiation
	 * @param selectedConverterType the converter type selected to write to the response
	 * @param request the current request
	 * @param response the current response
	 * @return the body that was passed in or a modified (possibly new) instance
	 */
	@Nullable
	T beforeBodyWrite(@Nullable T body, MethodParameter returnType, MediaType selectedContentType,
			Class<? extends HttpMessageConverter<?>> selectedConverterType,
			ServerHttpRequest request, ServerHttpResponse response);

}

```

也就是可以定制化通过`@ResponseBody`或者`ResponseEntity`配置的`controller`的返回值的`response`，而且是在生成`HttpMessageConverter`之前。
注意这里跟`HandlerInterceptorAdapter`的区别，`HandlerInterceptorAdapter`的`postHandler`方法是在生成`HttpMessageConverter`之后，才执行。
具体代码：
```Java
@RestControllerAdvice
public class HeaderModifierAdvice implements ResponseBodyAdvice {


    @Autowired
    ThreadLocal<Date> threadLocal;

    @Override
    public boolean supports(MethodParameter returnType, Class converterType) {
        return true;
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType,
                                  Class selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        response.getHeaders().add("cache-times",Long.toString(threadLocal.get().getTime()));
        return body;
    }
}
```
在前台的`Response Header`中：
```
Cache-Times: 1535552738328
```

可以发现，是在执行`controller`代码之后，被`beforeBodyWrite`方法注入`bean`之后，生成结果。
也正是这个问题，催生了这篇文章。


---

参考：
- [详述 Spring MVC 框架中拦截器 Interceptor 的使用方法][1]
- [Spring - Modify response headers after controller processing][3]

[1]:https://blog.csdn.net/qq_35246620/article/details/68487904?1491374806898
[2]:https://blog.csdn.net/zx13525079024/article/category/6214162/2
[3]:https://mtyurt.net/post/spring-modify-response-headers-after-processing.html
[4]:https://github.com/twentyworld/learn/tree/master/SpringBoot-Inteceptor-Filter
