# Callable

##### 为什么使用Callable
> 在Java中进行并行编程最常用的方式是继承Thread类或者实现Runnable接口。这两种方式的缺点是在任务完成后无法直接获取执行结果，必须通过共享变量或线程间通信，使用起来很不方便。
因为结果值是由另一线程计算的，当前线程是不知道结果值什么时候计算完成，所以它传递一个回调接口给计算线程，当计算完成时，调用这个回调接口，回传结果值。

> 所以，自从Java 1.5开始，就提供了Callable和Future，通过它们可以在任务执行完毕之后得到任务执行结果。Callable接口代表一段可以调用并返回结果的代码; Future接口表示异步任务，是还没有完成的任务给出的未来结果。**所以说Callable用于产生结果，Future用于获取结果。**


Callable接口类似于Runnable，从名字就可以看出来了，但是Runnable不会返回结果，并且无法抛出返回结果的异常，而Callable功能更强大一些，被线程执行后，可以返回值，这个返回值可以被Future拿到，也就是说，Future可以拿到异步执行任务的返回值。

Callable接口使用泛型去定义它的返回类型。Executors类提供了一些有用的方法在线程池中执行Callable内的任务。由于Callable任务是并行的，我们必须等待它返回的结果。java.util.concurrent.Future对象为我们解决了这个问题。在线程池提交Callable任务后返回了一个Future对象，使用它可以知道Callable任务的状态和得到Callable返回的执行结果。Future提供了get()方法让我们可以等待Callable结束并获取它的执行结果。


Callable接口位于java.util.concurrent包，这是一个泛型接口，里面只声明了一个call()方法：

```Java

/**
 * A task that returns a result and may throw an exception.
 * Implementors define a single method with no arguments called
 * {@code call}.
 *
 * <p>The {@code Callable} interface is similar to {@link
 * java.lang.Runnable}, in that both are designed for classes whose
 * instances are potentially executed by another thread.  A
 * {@code Runnable}, however, does not return a result and cannot
 * throw a checked exception.
 *
 * <p>The {@link Executors} class contains utility methods to
 * convert from other common forms to {@code Callable} classes.
 *
 * @see Executor
 * @since 1.5
 * @author Doug Lea
 * @param <V> the result type of method {@code call}
 */
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}

```
从接口的描述信息中，我们就可以发现很关键的信息。
> **A task that returns a result and may throw an exception.** Implementors define a single method with no arguments called call.
> The Callable interface is similar to Runnable, in that both are designed for classes whose instances are potentially executed by another thread. A Runnable, however, does not return a result and cannot throw a checked exception.
The Executors class contains utility methods to convert from other common forms to Callable classes.



```Java


```



```Java


```



```Java


```



```Java


```



```Java


```



```Java


```



```Java


```



```Java


```



```Java


```



```Java


```



```Java


```



```Java


```



```Java


```



```Java


```



```Java


```



```Java


```



```Java


```



```Java


```





































引用文章：

- [在Java中使用Callable、Future进行并行编程][1]
- [Java线程池_Future与Callable原理分析][2]




[1]:[https://segmentfault.com/a/1190000012291442]
[2]:[https://www.jianshu.com/p/fdef785bb287]
[3]:
[4]:
[5]:
[6]:
[7]:
[8]:
[9]:
[10]:
[11]:
[12]:
[13]:
[14]:
[15]:
[16]:
[17]:
[18]:
[19]:
