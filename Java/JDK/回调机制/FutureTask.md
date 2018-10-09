## FutureTask

FutureTask是一个可取消的异步计算任务。FutureTask提供了对 Future 的基本实现。仅在计算完成时才能获取结果；如果计算尚未完成，则阻塞 get 方法。一旦计算完成，就不能再重新开始或取消计算。

可使用 FutureTask 包装 Callable 或 Runnable 对象。因为 FutureTask 实现了 Runnable，所以可将 FutureTask 提交给 Executor 执行。

除了作为一个独立的类外，FutureTask还提供了 protected 功能（done()、set(V v)等方法），这在创建自定义任务类时可能很有用。

### Future的构造方法
```Java

public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}

public class FutureTask<V> implements RunnableFuture<V> {
  /**
   * Creates a {@code FutureTask} that will, upon running, execute the
   * given {@code Callable}.
   *
   * @param  callable the callable task
   * @throws NullPointerException if the callable is null
   */
  public FutureTask(Callable<V> callable) {
      if (callable == null)
          throw new NullPointerException();
      this.callable = callable;
      this.state = NEW;       // ensure visibility of callable
  }

  /**
   * Creates a {@code FutureTask} that will, upon running, execute the
   * given {@code Runnable}, and arrange that {@code get} will return the
   * given result on successful completion.
   *
   * @param runnable the runnable task
   * @param result the result to return on successful completion. If
   * you don't need a particular result, consider using
   * constructions of the form:
   * {@code Future<?> f = new FutureTask<Void>(runnable, null)}
   * @throws NullPointerException if the runnable is null
   */
  public FutureTask(Runnable runnable, V result) {
      this.callable = Executors.callable(runnable, result);
      this.state = NEW;       // ensure visibility of callable
  }

}
```
一种是直接指定 Callable为参数。另一种则是指定了 Runnable与 result。


### 1.  FutureTask的状态
根据Future接口，我们至少判断FutureTask有三种状态，新建状态，完成状态和取消状态。但是因为还有异常和中断操作，状态多了几种。
```Java
    // 表示FutureTask当前的状态
    private volatile int state;
    // NEW 新建状态，表示这个FutureTask还没有开始运行
    private static final int NEW          = 0;
    // COMPLETING 完成状态， 表示FutureTask任务已经计算完毕了，
    // 但是还有一些后续操作，例如唤醒等待线程操作，还没有完成。
    private static final int COMPLETING   = 1;
    // FutureTask任务完结，正常完成，没有发生异常
    private static final int NORMAL       = 2;
    // FutureTask任务完结，因为发生异常。
    private static final int EXCEPTIONAL  = 3;
    // FutureTask任务完结，因为取消任务
    private static final int CANCELLED    = 4;
    // FutureTask任务完结，也是取消任务，不过发起了中断运行任务线程的中断请求。
    private static final int INTERRUPTING = 5;
    // FutureTask任务完结，也是取消任务，已经完成了中断运行任务线程的中断请求。
    private static final int INTERRUPTED  = 6;
```


**状态说明：**
- NEW 新建状态: 是FutureTask刚创建时的状态，也就是说没有调用过这个FutureTask任务的run与runAndReset运行方法和cancel取消方法。

- COMPLETING 完成状态: 表示FutureTask任务的run运行方法已经执行完成，有两种方式表示完成：一是run方法正常执行完毕，二是run方法抛出异常。在set和setException方法中设置。

- NORMAL 正常完结状态：表示FutureTask任务顺利地执行完毕，得到运算结果，一般在set方法中设置。

- EXCEPTIONAL 异常完结状态：表示FutureTask任务在运行过程中抛出异常，任务也表示完结，一般在setException方法中设置。

- CANCELLED 取消状态：表示正常取消任务，但是不会对正在运行FutureTask任务的线程产生任何影响，也就是说FutureTask任务仍然会执行完毕，但是结果值无法设置了，因为任务已取消。通过调用cancel方法设置，参数mayInterruptIfRunning是false。

- INTERRUPTING和INTERRUPTED：表示中断取消任务，也就是说它会调用正在运行FutureTask任务的线程的interrupt方法，进行中断请求。通过调用cancel方法设置，参数mayInterruptIfRunning是true。

**这六种状态分为四类：**

- NEW状态：这个FutureTask任务还没有做任何操作，只有这个状态下的任务，我们可以调用run方法运行任务，或者调用cancel方法取消任务。

- COMPLETING状态：表示run方法运行完成，但是还有一些后序操作没有执行，比如唤醒正在等待任务结果的线程。切记这个状态FutureTask任务没有完结，不能返回结果值。

- NORMAL和EXCEPTIONAL状态：任务完结，可能是正常完结也可能是异常完结。

- CANCELLED、INTERRUPTING和INTERRUPTED状态：取消任务，任务也是完结。区别就是CANCELLED只是改变状态，而INTERRUPTING不仅改变状态，还会对正在运行FutureTask任务的线程进行中断。INTERRUPTED表示已经调用了中断请求。

>因为状态是int变量，所以可以根据大小来判断。state > COMPLETING：表示FutureTask任务已经完结了，可以返回结果值。state >= CANCELLED：表示任务已经取消了，也表示任务完结。


### 2.  FutureTask的获取结果值get方法
因为FutureTask任务一般运行在另一个线程中，其他线程来获取任务的结果值，会发生什么情况呢？
> 1.  如果FutureTask任务已经完结，那么就返回结果值.
> 2.  如果FutureTask任务没有完结，那么当前线程就应该等待，直到任务运行完结，会唤醒这个等待结果的线程，返回结果值。

```Java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    /**
     * 状态小于等于COMPLETING，表示FutureTask任务还没有完结，
     * 所以调用awaitDone方法，让当前线程等待
     */
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    // 返回结果值或者抛出异常
    return report(s);
}

public V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException {
    if (unit == null)
        throw new NullPointerException();
    int s = state;
    /**
     * 状态小于等于COMPLETING，表示FutureTask任务还没有完结，
     * 所以调用awaitDone方法，让当前线程等待。
     * 与get()不同的是，如果到了规定时间，任务状态仍然是小于等于COMPLETING，
     * 那么就抛出TimeoutException超时异常
     */
    if (s <= COMPLETING &&
        (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
        throw new TimeoutException();
    // 返回结果值或者抛出异常
    return report(s);
}

/**
 * 返回运行结果，或者抛出异常。这个两种情况都表示FutureTask完结了
 */
@SuppressWarnings("unchecked")
private V report(int s) throws ExecutionException {
    Object x = outcome;
    // 表示正常完结状态，所以返回结果值
    if (s == NORMAL)
        return (V)x;
    // 大于或等于CANCELLED，都表示手动取消FutureTask任务，
    // 所以抛出CancellationException异常
    if (s >= CANCELLED)
        throw new CancellationException();
    // 否则就是运行过程中，发生了异常，这里就抛出这个异常
    throw new ExecutionException((Throwable)x);
}


```
get方法流程与我们上面介绍的一样。s <= COMPLETING表示FutureTask任务没有完结，awaitDone方法可以将当前线程插入等待结果的线程队列中去，并阻塞当前线程。report方法返回结果值或者抛出异常。

report方法就是根据传入的状态值s，来决定是抛出异常，还是返回结果值。


### 3.  运行FutureTask任务
运行FutureTask任务也很简单，只要调用FutureTask任务的run方法，那么这个线程也是运行FutureTask任务的线程，取消任务时，可能会中断这个线程。

```Java
// 开始运行FutureTask任务
    public void run() {
        // 如果状态state不是NEW，或者设置runner值失败
        // 表示有别的线程在此之前调用run方法，并成功设置了runner值
        // 保证了只有一个线程可以运行try 代码块中的代码。
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            // 使用一个变量c记录callable，防止多线程情况下，
            // callable直接被设置为null出现问题
            Callable<V> c = callable;
            // 只有c不为null且状态state为NEW的情况，
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    // 调用callable的call方法，并返回结果
                    result = c.call();
                    // 运行成功
                    ran = true;
                } catch (Throwable ex) {
                    // 发生异常
                    result = null;
                    ran = false;
                    // 设置异常
                    setException(ex);
                }
                // 如果运行成功，则设置结果
                if (ran)
                    set(result);
            }
        } finally {
            runner = null;
            int s = state;
            // 当状态大于或等于INTERRUPTING，调用handlePossibleCancellationInterrupt方法，
            // 等待别的线程将状态设置成INTERRUPTED
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }

```
其实run方法作用非常简单，就是调用callable的call方法返回结果值result，根据是否发生异常，调用set(result)或setException(ex)方法表示FutureTask任务完结。

不过因为FutureTask任务都是在多线程环境中使用，所以要注意并发冲突问题。注意在run方法中，我们没有使用synchronized代码块或者Lock来解决并发问题，而是使用了CAS这个乐观锁来实现并发安全，保证只有一个线程能运行FutureTask任务。

```Java
protected void set(V v) {
    // 调用CAS函数，将状态state从NEW改成COMPLETING
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        // 延迟设置，将状态改成NORMAL，
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        // 调用finishCompletion唤醒所有等待结果的线程
        finishCompletion();
    }
}

protected void setException(Throwable t) {
    // 调用CAS函数，将状态state从NEW改成COMPLETING
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = t;
        // 延迟设置，将状态改成EXCEPTIONAL
        UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
        // 调用finishCompletion唤醒所有等待结果的线程
        finishCompletion();
    }
}
```
set与setException方法流程几乎一样：

1. 先使用CAS函数将状态state从NEW变成COMPLETING，防止多线程冲突。
2. 使用putOrderedInt方法设置状态state是NORMAL或EXCEPTIONAL。
3. 调用finishCompletion方法唤醒所有等待结果的线程。

> putOrderedInt方法的意义。因为state变量是被volatile关键字修饰，根据我们以前的讲解，那么它会给state变量加一个内存屏障，来保证state变量的可见性和有序性，这样会消耗一些性能。而putOrderedInt方法的意义，就是通过它来设置volatile修饰的变量，会取消这个内存屏障，也就是像普通变量一样了，不保证可见性了。



### 4.  取消FutureTask任务
```Java
public boolean cancel(boolean mayInterruptIfRunning) {
    /**
     * 如果当前状态不是NEW，或者使用CAS修改当前状态失败，那么直接返回false，取消失败
     */
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {
        // 如果能在运行时中断，那么就要调用运行FutureTask线程runner的interrupt方法
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    t.interrupt();
            } finally { // final state
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        // 调用finishCompletion唤醒等待结果的线程
        finishCompletion();
    }
    return true;
}
```    
使用CAS函数来修改state状态，保证并发问题。如果mayInterruptIfRunning为true，那么会调用正在运行FutureTask任务线程的interrupt方法，发起中断请求，最后调用finishCompletion方法唤醒等待结果的线程。



### 5.  等待结果的线程队列
```Java
/**
 * 简单地单向链表的节点。记录着所有等待FutureTask运行结果的线程
 */
static final class WaitNode {
    volatile Thread thread;
    // 下一个节点
    volatile WaitNode next;
    WaitNode() { thread = Thread.currentThread(); }
}

/** 单向链表，记录着所有等待FutureTask运行结果的线程 */
private volatile WaitNode waiters;
```

使用waiters来记录等待结果的线程队列。WaitNode内部类表示队列中的节点。


### 6.  将当前线程插入到等待队列中


```Java
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    // 计算截止日期
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    // 节点是否已添加
    boolean queued = false;
    for (;;) {
        // 如果当前线程中断标志位是true，
        // 那么从列表中移除节点q，并抛出InterruptedException异常
        if (Thread.interrupted()) {
            // 调用removeWaiter方法从链表中移除节点q
            removeWaiter(q);
            throw new InterruptedException();
        }

        int s = state;
        // 当状态大于COMPLETING时，表示FutureTask任务已结束。
        if (s > COMPLETING) {
            if (q != null)
                // 将节点q线程设置为null，因为线程没有阻塞等待
                q.thread = null;
            return s;
        }
        // 表示还有一些后序操作没有完成，那么当前线程让出执行权
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        // 代码来到这里，表示状态是NEW，那么就需要将当前线程阻塞等待。
        // 就是将它插入等待线程链表中，
        else if (q == null)
            // 使用当前线程创建节点p
            q = new WaitNode();
        //
        else if (!queued)
            // 使用CAS函数将新节点添加到链表中，如果添加失败，那么queued为false，
            // 下次循环时，会继续添加，知道成功。
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        // timed为true表示需要设置超时
        else if (timed) {
            // 得到剩余时间
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            // 让当前线程等待nanos时间
            LockSupport.parkNanos(this, nanos);
        }
        else
            // 让当前线程阻塞等待
            LockSupport.park(this);
    }
}
```

为了保证多线程安全问题，采用了CAS函数和for循环这种乐观锁的方法。

1. Thread.interrupted()返回true，当前线程发生了中断，那么从列表中移除节点q，并抛出InterruptedException异常。
2. s > COMPLETING表示FutureTask任务已经完结，不用阻塞当前线程了，直接返回状态。
3. 当s == COMPLETING：表示run方法已经执行完毕，还有一点后序操作没有完成，所以当前线程让出执行权，等待FutureTask任务状态变成完结状态。
4. 剩下的就表示状态是NEW，那么就用当前线程创建一个WaitNode节点，然后插入到等待线程队列waiters中，最后调用LockSupport的park系列方法将当前线程阻塞。
5. 被阻塞的线程等待着被唤醒，然后进行下一次for循环，继续判断。
```Java
// 从链表中删除节点node
private void removeWaiter(WaitNode node) {
    if (node != null) {
        // 将thread设置null
        node.thread = null;
        retry:
        for (;;) {          // restart on removeWaiter race
            for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                // 记录当前节点q的下一个节点s
                s = q.next;
                // 如果当前节点q的thread不等于null，那么就用pred记录q
                if (q.thread != null)
                    pred = q;
                // 如果当前节点q的thread等于null，且pred不等于null
                else if (pred != null) {
                    // 删除当前节点q
                    pred.next = s;
                    // 如果pred.thread == null，那么继续retry的for循环
                    if (pred.thread == null) // check for race
                        continue retry;
                }
                else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                      q, s))
                    continue retry;
            }
            break;
        }
    }
}
```
### 7.  唤醒所有等待结果的线程
```Java
/**
 * 当FutureTask任务结束时(包括运行完成、抛出异常以及手动取消都表示任务结束)，都会调用这个方法。
 * 用来唤醒所有等待运行结果的线程。
 */
private void finishCompletion() {
    // assert state > COMPLETING;
    // 这里使用CAS函数实现的乐观锁，保证只有一个线程能循环单向链表waiters
    for (WaitNode q; (q = waiters) != null;) {
        // 如果返回false，表示waiters被别的线程更改了，那么就再次循环。
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            // 循环等待结果的线程链表
            for (;;) {
                Thread t = q.thread;
                // t不为null，那么就要唤醒等待线程
                if (t != null) {
                    q.thread = null;
                    LockSupport.unpark(t);
                }
                // 下一个节点
                WaitNode next = q.next;
                // next == null表示遍历到链表尾了。
                if (next == null)
                    break;
                // help gc
                q.next = null;
                q = next;
            }
            break;
        }
    }
    // 钩子方法。子类可以复写这个方法。
    done();
    callable = null;        // to reduce footprint
}
```

从上面的分析，这个方法会在set()、setException()和cancel()方法中调用。
遍历等待结果的线程队列waiters，然后通过LockSupport.unpark(t)方法，唤醒被阻塞的线程。


### 8.  重复运行FutureTask任务
```Java
 // 可以重复运行FutureTask任务
protected boolean runAndReset() {
    // 如果状态state不是NEW，或者设置runner值失败
    // 表示有别的线程在此之前调用run方法，并成功设置了runner值
    // 保证了只有一个线程可以运行try 代码块中的代码。
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return false;
    boolean ran = false;
    int s = state;
    try {
        Callable<V> c = callable;
        if (c != null && s == NEW) {
            try {
                // 注意这里没有获取结果值
                // 因为任务可以重复执行，所以任务状态必须还是NEW。
                // 不能调用set(result)方法改变任务状态
                c.call();
                ran = true;
            } catch (Throwable ex) {
                setException(ex);
            }
        }
    } finally {
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
    // 只有call方法调用成功且任务状态还是NEW，才可以再次执行任务
    return ran && s == NEW;
}
```
runAndReset方法与run方法的区别就是当任务运行完毕后，不会调用set(result)方法，设置任务的结果值，改变任务的状态。所以可以重复调用runAndReset方法来多次运行任务。主要是为了定时线程池中，可以重复周期性地运行任务。

### 总结
Callable表示一个任务，当任务执行完成后会返回结果值。但是结果值并不是立即返回的，那么其他线程怎么获取这个结果值呢？那么就需要Future对象。
Future就是监控Callable任务完成情况的，通过组合的方式，即Future对象实例一般有个Callable实例的成员变量。当Callable任务还没有完成时，调用Future的get方法，就会让当前线程等待，直到Callable任务完成。

### 重要示例

```Java
import java.util.concurrent.*;
import java.util.concurrent.Callable;

public class FutureTaskTest {

    public static void newThread(String name, Future<Integer> future) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("线程"+Thread.currentThread().getName()+"开始运行");
                try {
                    int result = future.get();
                    System.out.println("线程"+Thread.currentThread().getName()+"获取结果  result=="+result);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (ExecutionException e) {
                    e.printStackTrace();
                }
            }
        }, name).start();
    }

    public static void main(String[] args)  {
        FutureTask<Integer> future = new FutureTask<Integer>(new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                System.out.println("线程"+Thread.currentThread().getName()+"运行任务");
                Thread.sleep(1000);
                System.out.println("线程"+Thread.currentThread().getName()+"任务运行完成");
                return 100;
            }
        });
        new Thread(new Runnable() {
            @Override
            public void run() {
                future.run();
            }
        }, "t10").start();

        newThread("t1", future);
        newThread("t2", future);
        newThread("t3", future);
        newThread("t4", future);
    }
}
```

---
参考文章：
- [Java线程池_Future与Callable原理分析][1]

[1][https://www.jianshu.com/p/fdef785bb287]
