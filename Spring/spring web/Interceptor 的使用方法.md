#Interceptor的使用方法

>本文全文copy自作者`维C果糖`,文章出自`CSDN`。本文仅做学习记录使用。
>如果想更详细的了解文章，请[点击][1]去原文。表达对博主的感谢。


##1.拦截器概念
`Java`里的拦截器是动态拦截`action`调用的对象。它提供了一种机制可以使开发者可以定义在一个`action`执行的前后执行的代码，也可以在一个`action`执行前阻止其执行，同时也提供了一种可以提取`action`中可重用部分的方式。在`AOP`（`Aspect-OrientedProgramming`，面向切面编程）中拦截器用于在某个方法或字段被访问之前进行拦截，然后在之前或之后加入某些操作。


##2.实现方法

在Spring框架之中，咱们要想实现拦截器的功能，主要通过两种途径，第一种是实现HandlerInterceptor接口，第二种是实现WebRequestInterceptor接口。接下来，咱们分别详细的介绍两者的实现方法。

###2.1HandlerInterceptor接口
在HandlerInterceptor接口中，定义了3个方法，分别为preHandle()、postHandle()和afterCompletion()，咱们就是通过复写这3个方法来对用户的请求进行拦截处理的。因此，咱们可以通过直接实现HandlerInterceptor接口来实现拦截器的功能。不过在Spring框架之中，其还提供了另外一个接口和一个抽象类，实现了对HandlerInterceptor接口的功能扩展，分别为：AsyncHandlerInterceptor和HandlerInterceptorAdapter.

对于AsyncHandlerInterceptor接口，其在继承HandlerInterceptor接口的同时，又声明了一个新的方法afterConcurrentHandlingStarted()；而HandlerInterceptorAdapter抽象类，则是更进一步，在其继承AsyncHandlerInterceptor接口的同时，又复写了preHandle方法。因此，AsyncHandlerInterceptor更像是一个过渡的接口。

在实际应用中，咱们一般都是通过实现HandlerInterceptor接口或者继承HandlerInterceptorAdapter抽象类，复写preHandle()、postHandle()和afterCompletion()这3个方法来对用户的请求进行拦截处理的。下面，咱们就详细介绍这个3个方法。


preHandle(HttpServletRequestrequest,HttpServletResponseresponse,Objecthandle)方法，该方法在请求处理之前进行调用。SpringMVC中的Interceptor是链式调用的，在一个应用中或者说是在一个请求中可以同时存在多个Interceptor。每个Interceptor的调用会依据它的声明顺序依次执行，而且最先执行的都是Interceptor中的preHandle方法，所以可以在这个方法中进行一些前置初始化操作或者是对当前请求做一个预处理，也可以在这个方法中进行一些判断来决定请求是否要继续进行下去。该方法的返回值是布尔值Boolean类型的，当它返回为false时，表示请求结束，后续的Interceptor和Controller都不会再执行；当返回值为true时，就会继续调用下一个Interceptor的preHandle方法，如果已经是最后一个Interceptor的时候，就会是调用当前请求的Controller中的方法。

postHandle(HttpServletRequestrequest,HttpServletResponseresponse,Objecthandle,ModelAndViewmodelAndView)方法，通过preHandle方法的解释咱们知道这个方法包括后面要说到的afterCompletion方法都只能在当前所属的Interceptor的preHandle方法的返回值为true的时候，才能被调用。postHandle方法在当前请求进行处理之后，也就是在Controller中的方法调用之后执行，但是它会在DispatcherServlet进行视图返回渲染之前被调用，所以咱们可以在这个方法中对Controller处理之后的ModelAndView对象进行操作。postHandle方法被调用的方向跟preHandle是相反的，也就是说，先声明的Interceptor的postHandle方法反而会后执行。这和Struts2里面的Interceptor的执行过程有点类型，Struts2里面的Interceptor的执行过程也是链式的，只是在Struts2里面需要手动调用ActionInvocation的invoke方法来触发对下一个Interceptor或者是action的调用，然后每一个Interceptor中在invoke方法调用之前的内容都是按照声明顺序执行的，而invoke方法之后的内容就是反向的。

afterCompletion(HttpServletRequestrequest,HttpServletResponseresponse,Objecthandle,Exceptionex)方法，也是需要当前对应的Interceptor的preHandle方法的返回值为true时才会执行。因此，该方法将在整个请求结束之后，也就是在DispatcherServlet渲染了对应的视图之后执行，这个方法的主要作用是用于进行资源清理的工作。


HandlerInterceptor接口

```Java
publicinterfaceHandlerInterceptor{

	boolean preHandle(HttpServletRequestrequest,HttpServletResponseresponse,Objecthandler)
			throwsException;

	voidpostHandle(
			HttpServletRequestrequest,HttpServletResponseresponse,Objecthandler,ModelAndViewmodelAndView)
			throwsException;

	voidafterCompletion(
			HttpServletRequestrequest,HttpServletResponseresponse,Objecthandler,Exceptionex)
			throwsException;

}

```

AsyncHandlerInterceptor 接口
```Java

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public interface AsyncHandlerInterceptor extends HandlerInterceptor {

    void afterConcurrentHandlingStarted(
            HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception;
}
```
HandlerInterceptorAdapter 抽象类：

```Java
package org.springframework.web.servlet.handler;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.springframework.web.servlet.AsyncHandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

/**
 * Abstract adapter class for the HandlerInterceptor interface,
 * for simplified implementation of pre-only/post-only interceptors.
 *
 * @author Juergen Hoeller
 * @since 05.12.2003
 */
public abstract class HandlerInterceptorAdapter implements AsyncHandlerInterceptor {

    /**
     * This implementation always returns {@code true}.
     */
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
        throws Exception {
        return true;
    }

    /**
     * This implementation is empty.
     */
    public void postHandle(
            HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)
            throws Exception {
    }

    /**
     * This implementation is empty.
     */
    public void afterCompletion(
            HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
            throws Exception {
    }

    /**
     * This implementation is empty.
     */
    public void afterConcurrentHandlingStarted(
            HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
    }
}
```

如上面的代码所示，其实在HandlerInterceptor和AsyncHandlerInterceptor中还有很多的代码注释，只是博主感觉太多了，就将其全部删除啦！如果大家对这些注释感兴趣的话，可以自行查看源代码。下面，咱们以继承HandlerInterceptorAdapter抽象类为例进行演示：

```Java

```

###2.2 WebRequestInterceptor 接口
在WebRequestInterceptor接口中也定义了 3 个方法，同HandlerInterceptor接口完全相同，咱们也是通过复写这 3 个方法来用户的请求进行拦截处理的。而且这 3 个方法都传递了同一个参数 WebRequest，那么这个 WebRequest 到底是什么呢？其实这个 WebRequest 是 Spring 中定义的一个接口，它里面的方法定义跟 HttpServletRequest 类似，在WebRequestInterceptor中对 WebRequest 进行的所有操作都将同步到 HttpServletRequest 中，然后在当前请求中依次传递。

在 Spring 框架之中，还提供了一个和WebRequestInterceptor接口长的很像的抽象类，那就是：WebRequestInterceptorAdapter，其实现了AsyncHandlerInterceptor接口，并在内部调用了WebRequestInterceptor接口。

接下来，咱们主要讲一下WebRequestInterceptor接口的 3 个函数

preHandle(WebRequest request)方法，该方法在请求处理之前进行调用，也就是说，其会在 Controller 中的方法调用之前被调用。这个方法跟 HandlerInterceptor 中的 preHandle 不同，主要区别在于该方法的返回值是void 类型的，也就是没有返回值，因此我们主要用它来进行资源的准备工作，比如我们在使用 Hibernate 的时候，可以在这个方法中准备一个 Hibernate 的Session 对象，然后利用 WebRequest 的 setAttribute(name, value, scope) 把它放到 WebRequest 的属性中。在这里，进一步说说 setAttribute 方法的第三个参数 scope ，该参数是一个Integer 类型的。在 WebRequest 的父层接口 RequestAttributes 中对它定义了三个常量，分别为：
SCOPE_REQUEST ，它的值是 0，表示只有在 request 中可以访问。
SCOPE_SESSION，它的值是1，如果环境允许的话，它表示的是一个局部的隔离的 session，否则就代表普通的 session，并且在该 session 范围内可以访问。
SCOPE_GLOBAL_SESSION，它的值是 2，如果环境允许的话，它表示的是一个全局共享的 session，否则就代表普通的 session，并且在该 session 范围内可以访问。
postHandle(WebRequest request, ModelMap model)方法，该方法在请求处理之后，也就是在 Controller 中的方法调用之后被调用，但是会在视图返回被渲染之前被调用，所以可以在这个方法里面通过改变数据模型 ModelMap 来改变数据的展示。该方法有两个参数，WebRequest 对象是用于传递整个请求数据的，比如在 preHandle 中准备的数据都可以通过 WebRequest 来传递和访问；ModelMap 就是 Controller 处理之后返回的 Model 对象，咱们可以通过改变它的属性来改变返回的 Model 模型。
afterCompletion(WebRequest request, Exception ex)方法，该方法会在整个请求处理完成，也就是在视图返回并被渲染之后执行。因此可以在该方法中进行资源的释放操作。而 WebRequest 参数就可以把咱们在 preHandle 中准备的资源传递到这里进行释放。Exception 参数表示的是当前请求的异常对象，如果在 Controller 中抛出的异常已经被 Spring 的异常处理器给处理了的话，那么这个异常对象就是是 null.




### 2.3 AbstractInterceptor 抽象类
```Java

```


```Java

```


```Java

```




















































[1]:https://blog.csdn.net/qq_35246620/article/details/68487904?1491374806898
[2]:https://blog.csdn.net/zx13525079024/article/category/6214162/2
[3]:https://mtyurt.net/post/spring-modify-response-headers-after-processing.html
