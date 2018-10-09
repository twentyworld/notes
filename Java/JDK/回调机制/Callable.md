# Callable

##### 1. 为什么使用Callable

> 在Java中进行并行编程最常用的方式是继承Thread类或者实现Runnable接口。这两种方式的缺点是在任务完成后无法直接获取执行结果，必须通过共享变量或线程间通信，使用起来很不方便。
因为结果值是由另一线程计算的，当前线程是不知道结果值什么时候计算完成，所以它传递一个回调接口给计算线程，当计算完成时，调用这个回调接口，回传结果值。

> 所以，自从Java 1.5开始，就提供了Callable和Future，通过它们可以在任务执行完毕之后得到任务执行结果。Callable接口代表一段可以调用并返回结果的代码; Future接口表示异步任务，是还没有完成的任务给出的未来结果。**所以说Callable用于产生结果，Future用于获取结果。**


Callable接口使用泛型去定义它的返回类型。Executors类提供了一些有用的方法在线程池中执行Callable内的任务。由于Callable任务是并行的，我们必须等待它返回的结果。java.util.concurrent.Future对象为我们解决了这个问题。在线程池提交Callable任务后返回了一个Future对象，使用它可以知道Callable任务的状态和得到Callable返回的执行结果。Future提供了get()方法让我们可以等待Callable结束并获取它的执行结果。

### 2. Callable和Runnable
- Callable接口类似于Runnable，从名字就可以看出来了，但是Runnable不会返回结果，并且无法抛出返回结果的异常，而Callable功能更强大一些，被线程执行后，可以返回值。

- Callable是 java.util.concurrent中的接口，而 Runnable是 java.lang包中的接口。它们都只有一个方法。

-  Callable并不像Runnable那样通过Thread的start方法就能启动实现类的run方法，所以它通常利用ExecutorService的submit方法去启动call方法自执行任务，而ExecutorService的submit又返回一个Future类型的结果，因此Callable通常也与Future一起使用

或者利用FutureTask封装Callable再由Thread去启动：
```Java
FutureTask<String> task = new FutureTask(new Callable{
       public void call(){
             //TODO
       }
 });
Thead thread = new Thread(task);
thread.start();
```
> 但是很不常用。

#### 2.1 Callable转换
Executors 类包含一些从其他普通形式（Runnable、PrivilegedAction等）转换成 Callable 类的实用方法。
```Java
/**
 * 返回 Callable 对象，调用它时可运行给定的任务并返回 null
 */  
static Callable<Object> callable(Runnable task)   
/**
 * 返回 Callable 对象，调用它时可运行给定的任务并返回给定的结果
 */  
static <T> Callable<T> callable(Runnable task, T result)   
```
我们可以通过 Executors类的callable方法将 Runnable 转换成 Callable，其中 result为返回结果，以下是callable方法的源代码

```Java
 public static <T> Callable<T> callable(Runnable task, T result) {  
    if (task == null)  
        throw new NullPointerException();  
    return new RunnableAdapter<T>(task, result);  
}  

/**
 * A callable that runs given task and returns given result
 */
static final class RunnableAdapter<T> implements Callable<T> {
    final Runnable task;
    final T result;
    RunnableAdapter(Runnable task, T result) {
        this.task = task;
        this.result = result;
    }
    public T call() {
        task.run();
        return result;
    }
}

```
> 但是这个结果并不是Runnable的执行结果(Runnable的run方法是void类型)，而是执行者预定义的结果，这点可以从其实现原理RunnableAdpter源码看出

当 task为空时会抛出空指针异常，然后通过构造方法创建一个 RunnableAdapter实例。RunnableAdapter类是一个内部类，其实 RunnableAdapter是一个 Runnable的适配器，它的作用很简单，就是将 Runnable转换成 Callable，所以这里就用到了适配器模式. 代码即简洁有清晰，通过调用构造方法创建一个 RunnableAdapter实例，然后通过执行call方法运行线程，最后返回结果。

### 3.  Callable源码

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



### 4.  Callable的使用
因为 Callable 接口并没有相关实现类，所以我们无法直接使用它，所以使用 Callable 需要调用 ExecutorService接口的 submit方法
```Java
<T> Future<T> submit(Callable<T> task)  
```
submit提交一个返回值的任务用于执行，返回一个表示任务的未决结果的 Future。该 Future 的 get 方法在成功完成时将会返回该任务的结果。

```Java
result = executorService.submit(aCallable).get();   
```

> Executors 类包括了一组方法，可以转换某些其他常见的类似于闭包的对象，例如，将 PrivilegedAction 转换为 Callable 形式，这样就可以提交它们了。

#### 4.1 Callable 异步原理简析
Callable又是如何实现的异步的呢，下面我们循序渐进，慢慢分析。 先看一个例子，实现Callable接口，进行异步计算：

```Java
public class Demo {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService executor = Executors.newCachedThreadPool();
        Future<String> future = executor.submit(new Callable<String>() {
            @Override
            public String call() throws Exception {
                System.out.println("call");
                TimeUnit.SECONDS.sleep(1);
                return "str";
            }
        });
        System.out.println(future.get());
    }
}
```
在不阻塞当前线程的情况下计算，那么必然需要另外的线程去执行具体的业务逻辑，上面代码中可以看到，是把Callable放入了线程池中，等待执行，并且立刻返回futrue。可以猜想下，需要从Future中得到Callable的结果，那么Future的引用必然会被两个线程共享，一个线程执行完成后改变Future的状态位并唤醒挂起在get上的线程，到底是不是这样呢？

##### 源码分析
```Java
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}

protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}

public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();

        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }


```

可以看到Callable任务被包装成了RunnableFuture对象，通过了线程池的execute方法提交任务并且立刻返回对象本身，而线程池是接受Runnable. 所以这里使用的是FutureTask，FutureTask实现了RunnableFuture， 而RunnableFuture继承了Runnable,Future，又接受Callable作为参数声称对象<- 通过构造方法。

到这里我们应该有些头绪了，关键点应该在FutureTask对象上，线程池不过是提供一个线程运行FutureTask中的run方法罢了。

从上面的分析，FutureTask被生产者和消费者共享，生产者运行run方法计算结果，消费者通过get方法获取结果，那么必然就需要通知，如何通知呢，肯定是状态位变化，并唤醒线程。

所以，这里我们基本可以确定的是
1. 线程池是生产者，他们通过执行FutureTask的run()方法，来生产结果。
2. 消费者是客户端，通过get()方法来的到结果。

如果想了解更多关于[FutureTask][5]的问题，请点击前往。




---



引用文章：

- [在Java中使用Callable、Future进行并行编程][1]
- [Java线程池_Future与Callable原理分析][2]
- [Java并发之Callable,Future,FutureTask][3]
- [Callable异步原理简析][4]


[1]:[https://segmentfault.com/a/1190000012291442]
[2]:[https://www.jianshu.com/p/fdef785bb287]
[3]:http://286.iteye.com/blog/2299701
[4]:https://blog.csdn.net/u012664375/article/details/66967687
[5]:FutureTask.md
