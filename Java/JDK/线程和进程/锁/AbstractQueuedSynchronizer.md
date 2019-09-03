AbstractQueuedSynchronizer

首先我们可以看看类的定义：
Provides a framework for implementing blocking locks and related synchronizers (semaphores, events, etc) that rely on **first-in-first-out (FIFO) wait queues**. This class is designed to be a useful basis for most kinds of synchronizers that rely on a single atomic int value to represent state. Subclasses must define the protected methods that change this state, and which define what that state means in terms of this object being acquired or released. Given these, the other methods in this class carry out all queuing and blocking mechanics. Subclasses can maintain other state fields, but only the atomically updated int value manipulated using methods getState(), setState(int) and compareAndSetState(int, int) is tracked with respect to synchronization.

**Subclasses should be defined as non-public internal helper classes that are used to implement the synchronization properties of their enclosing class.** Class AbstractQueuedSynchronizer does not implement any synchronization interface. Instead it defines methods such as acquireInterruptibly(int) that can be invoked as appropriate by concrete locks and related synchronizers to implement their public methods.

**This class supports either or both a default exclusive mode and a shared mode.** When acquired in exclusive mode, attempted acquires by other threads cannot succeed. Shared mode acquires by multiple threads may (but need not) succeed. This class does not "understand" these differences except in the mechanical sense that when a shared mode acquire succeeds, the next waiting thread (if one exists) must also determine whether it can acquire as well. Threads waiting in the different modes share the same FIFO queue. Usually, implementation subclasses support only one of these modes, but both can come into play for example in a ReadWriteLock. Subclasses that support only exclusive or only shared modes need not define the methods supporting the unused mode.

**This class defines a nested AbstractQueuedSynchronizer.ConditionObject class that can be used as a Condition implementation by subclasses supporting exclusive mode for which method isHeldExclusively() reports whether synchronization is exclusively held with respect to the current thread,** method release(int) invoked with the current getState() value fully releases this object, and acquire(int), given this saved state value, eventually restores this object to its previous acquired state. No AbstractQueuedSynchronizer method otherwise creates such a condition, so if this constraint cannot be met, do not use it. The behavior of AbstractQueuedSynchronizer.ConditionObject depends of course on the semantics of its synchronizer implementation.

This class provides inspection, instrumentation, and monitoring methods for the internal queue, as well as similar methods for condition objects. These can be exported as desired into classes using an AbstractQueuedSynchronizer for their synchronization mechanics.

Serialization of this class stores only the underlying atomic integer maintaining state, so deserialized objects have empty thread queues. Typical subclasses requiring serializability will define a readObject method that restores this to a known initial state upon deserialization.

---

## 1. 背景知识

## 2. 字段解析

先来看看 `AQS` 有哪些属性
```java

  // 头结点，你直接把它当做 当前持有锁的线程
  private transient volatile Node head;

  // 阻塞的尾节点，每个新的节点进来，都插入到最后，也就形成了一个链表
  private transient volatile Node tail;

  //共享资源 由子类维护
  private volatile int state;

  // 代表当前持有独占锁的线程，举个最重要的使用例子，因为锁可以重入
  // reentrantLock.lock()可以嵌套调用多次，所以每次用这个来判断当前线程是否已经拥有了锁
  // if (currentThread == getExclusiveOwnerThread()) {state++}
  private transient Thread exclusiveOwnerThread; //继承自AbstractOwnableSynchronizer
```

它维护了一个`volatile int state`（代表共享资源）和一个`FIFO`线程等待队列（多线程争用资源被阻塞时会进入此队列）。这里`volatile`是核心关键词，具体`volatile`的语义，在此不述。

### 2.1 共享资源 state

state的访问方式有三种:

> - getState()
> - setState()
> - compareAndSetState()

`AQS`定义两种资源共享方式：`Exclusive` (独占，只有一个线程能执行，如`ReentrantLock`)和`Share`(共享，多个线程可同时执行，如`Semaphore`/`CountDownLatch`)。

不同的自定义同步器争用共享资源的方式也不同。**自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），`AQS`已经在顶层实现好了。** 自定义同步器实现时主要实现以下几种方法：

> - isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。
> - tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false。
> - tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败则返回false。
> - tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
> - tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

以`ReentrantLock`为例，`state`初始化为`0`，表示未锁定状态。`A`线程`lock()`时，会调用`tryAcquire()`独占该锁并将`state+1`。此后，其他线程再`tryAcquire()`时就会失败，直到`A`线程`unlock()`到`state=0`(即释放锁)为止，其它线程才有机会获取该锁。当然，释放锁之前，`A`线程自己是可以重复获取此锁的(`state`会累加)，这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证`state`是能回到零态的。

再以`CountDownLatch`以例，任务分为`N`个子线程去执行，`state`也初始化为`N`(注意`N`要与线程个数一致)。这`N`个子线程是并行执行的，每个子线程执行完后`countDown()`一次，`state`会`CAS`减`1`。等到所有子线程都执行完后(即`state=0`)，会`unpark()`主调用线程，然后主调用线程就会从`await()`函数返回，继续后余动作。

一般来说，自定义同步器要么是独占方法，要么是共享方式，他们也只需实现`tryAcquire-tryRelease`、`tryAcquireShared-tryReleaseShared`中的一种即可。但`AQS`也支持自定义同步器同时实现独占和共享两种方式，如`ReentrantReadWriteLock`。

下图展示了，队列中的线程如何竞争共享资源：

![node-acquire-state][2]


#### 2.1.1 原子操作
`AQS`中对`state`的操作是原子的，且不能被继承。所有的同步机制的实现均依赖于对改变量的原子操作。 `AbstractQueuedSynchronizer`维护了一个`volatile int`类型的`state`，用户表示当前同步状态。`volatile`虽然不能保证操作的原子性，但是保证了当前变量`state`的可见性。

 上文中的`state`的三个方法都是原子操作，其中`compareAndSetState`的实现依赖于`Unsafe`的`compareAndSwapInt()`方法。

```java
  /**
   * The synchronization state.
   */
  private volatile int state;

  /**
   * Returns the current value of synchronization state.
   * This operation has memory semantics of a {@code volatile} read.
   * @return current state value
   */
  protected final int getState() {
      return state;
  }

  /**
   * Sets the value of synchronization state.
   * This operation has memory semantics of a {@code volatile} write.
   * @param newState the new state value
   */
  protected final void setState(int newState) {
      state = newState;
  }

  /**
   * Atomically sets synchronization state to the given updated
   * value if the current state value equals the expected value.
   * This operation has memory semantics of a {@code volatile} read
   * and write.
   *
   * @param expect the expected value
   * @param update the new value
   * @return {@code true} if successful. False return indicates that the actual
   *         value was not equal to the expected value.
   */
  protected final boolean compareAndSetState(int expect, int update) {
      // See below for intrinsics setup to support this
      return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
  }
```



### 2.2 线程阻塞队列 Node

等待队列中每个线程被包装成一个 `Node` 实例，数据结构是链表。

```java
static final class Node {
    // 标识节点当前在共享模式下
    static final Node SHARED = new Node();
    // 标识节点当前在独占模式下
    static final Node EXCLUSIVE = null;

    // ======== 下面的几个int常量是给waitStatus用的 ===========
    /** waitStatus value to indicate thread has cancelled */
    // 代码此线程取消了争抢这个锁
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking */
    // 官方的描述是，其表示当前node的后继节点对应的线程需要被唤醒
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition */
    // 本文不分析condition，所以略过吧，下一篇文章会介绍这个
    static final int CONDITION = -2;
    /**
     * waitStatus value to indicate the next acquireShared should
     * unconditionally propagate
     */
    // 同样的不分析，略过吧
    static final int PROPAGATE = -3;
    // =====================================================


    // 取值为上面的1、-1、-2、-3，或者0
    // 这么理解，暂时只需要知道如果这个值 大于0 代表此线程取消了等待，
    //    ps: 半天抢不到锁，不抢了，ReentrantLock是可以指定timeouot的。。。
    volatile int waitStatus;
    // 前驱节点的引用
    volatile Node prev;
    // 后继节点的引用
    volatile Node next;
    // 这个就是线程本尊
    volatile Thread thread;
}
```

需要强调，阻塞队列不包含 `head` 节点。

![image][1]



## 3. 源码实现
接下来我们开始开始讲解`AQS`的源码实现。依照`acquire-release`、`acquireShared-releaseShared`的次序来。

### 3.1 独占式AQS

#### 3.1.1 获取资源 acquire(int)

此方法是独占模式下线程获取共享资源的顶层入口。如果获取到资源，线程直接返回，否则进入等待队列，直到获取到资源为止，且整个过程忽略中断的影响。这也正是`lock()`的语义，当然不仅仅只限于`lock()`。获取到资源后，线程就可以去执行其临界区代码了。下面是`acquire()`的源码：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

函数流程如下：
1. `tryAcquire()`尝试直接去获取资源，如果成功则直接返回。 注意，这里的`tryAcquire`是由用户自定义。
2. `addWaiter()`将该线程加入等待队列的尾部，并标记为独占模式。
3. `acquireQueued()`使线程在等待队列中获取资源，一直获取到资源后才返回。如果在整个等待过程中被中断过，则返回`true`，否则返回`false`。
4. 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断`selfInterrupt()`，将中断补上。

##### 3.1.1.1 tryAcquire(int)

`AQS`只是一个框架，具体资源的获取/释放方式交由自定义同步器就是在这里实现，`AQS`这里只定义了一个接口，具体资源的获取交由自定义同步器去实现了(通过`state`的`get/set/CAS`). 至于能不能重入，能不能加塞(公平与非公平)，那就看具体的自定义同步器怎么去设计。当然，自定义同步器在进行资源访问时要考虑线程安全的影响。

之所以没有定义成`abstract`，是因为独占模式下只用实现`tryAcquire-tryRelease`，而共享模式下只用实现`tryAcquireShared-tryReleaseShared`。如果都定义成`abstract`，那么每个模式也要去实现另一模式下的接口。`Doug Lea`从开发者的角度，尽量减少不必要的工作量。

##### 3.1.1.2 addWaiter(Node)
此方法用于将当前线程加入到等待队列的队尾，并返回当前线程所在的结点。

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```
这里，可以看出，首先直接通过`compareAndSetTail`先尝试一下能不能直接把节点插入到尾节点成功，如果成功了，就直接放回。如果不成功，就是进入`enq`方法. 如果`tail` 节点是`null`， 也直接交由`enq`方法来处理。

**enq(final Node node)**

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
这里，很有趣的就是，通过`for`循环，一直尝试插入。**`CAS`自旋`volatile`变量**，是一种很经典的用法。

##### 3.1.1.3 acquireQueued(Node, int)
 `acquireQueued()`用于队列中的线程自旋地以独占且不可中断的方式获取同步状态（acquire），直到拿到锁之后再返回。该方法的实现分成两部分：如果当前节点已经成为头结点，尝试获取锁（tryAcquire）成功，然后返回；否则检查当前节点是否应该被`park`，然后将该线程`park`并且检查当前线程是否被可以被中断。
```java
  /**
   * Acquires in exclusive uninterruptible mode for thread already in
   * queue. Used by condition wait methods as well as acquire.
   *
   * @param node the node
   * @param arg the acquire argument
   * @return {@code true} if interrupted while waiting
   */
  final boolean acquireQueued(final Node node, int arg) {
      //标记是否成功拿到资源，默认false
      boolean failed = true;
      try {
          boolean interrupted = false;//标记等待过程中是否被中断过
          for (;;) {
              final Node p = node.predecessor();
              if (p == head && tryAcquire(arg)) {
                  setHead(node);
                  p.next = null; // help GC
                  failed = false;
                  return interrupted;
              }
              if (shouldParkAfterFailedAcquire(p, node) &&
                  parkAndCheckInterrupt())
                  interrupted = true;
          }
      } finally {
          if (failed)
              cancelAcquire(node);
      }
  }
```
从整体来看，这又是一个自选过程。代码首先获取当前节点的前驱节点，如果先驱节点是头结点的并且成功获得同步状态的时候`if (p == head && tryAcquire(arg))`，当前节点所指向的线程能够获取锁。反之，获取锁失败进入等待状态。

**获取锁成功，出队操作**

获取锁的节点出队的逻辑是：
```java
//队列头结点引用指向当前节点
setHead(node);
//释放前驱节点
p.next = null; // help GC
failed = false;
return interrupted;
```

将当前节点通过`setHead()`方法设置为队列的头结点，然后将之前的头结点的`next`域设置为`null`并且`pre`域也为`null`，即与队列断开，无任何引用方便`GC`时能够将内存进行回收。
示意图如下：

![获取资源成功][4]

那么当获取锁失败的时候会调用`shouldParkAfterFailedAcquire()`方法和`parkAndCheckInterrupt()`方法。

我们可以先看看`shouldParkAfterFailedAcquire()`和`parkAndCheckInterrupt()`具体干些什么。

**shouldParkAfterFailedAcquire(Node, Node)**

 `shouldParkAfterFailedAcquire`方法通过对当前节点的前一个节点的状态进行判断，对当前节点做出不同的操作，至于每个`Node`的状态表示，可以参考接口文档。

```java
  /**
   * Checks and updates status for a node that failed to acquire.
   * Returns true if thread should block. This is the main signal
   * control in all acquire loops.  Requires that pred == node.prev.
   *
   * @param pred node's predecessor holding status
   * @param node the node
   * @return {@code true} if thread should block
   */
  private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
      int ws = pred.waitStatus;
      if (ws == Node.SIGNAL)
          /*
           * This node has already set status asking a release
           * to signal it, so it can safely park.
           */
          return true;
      if (ws > 0) {
          /*
           * Predecessor was cancelled. Skip over predecessors and
           * indicate retry.
           */
          do {
              node.prev = pred = pred.prev;
          } while (pred.waitStatus > 0);
          pred.next = node;
      } else {
          /*
           * waitStatus must be 0 or PROPAGATE.  Indicate that we
           * need a signal, but don't park yet.  Caller will need to
           * retry to make sure it cannot acquire before parking.
           */
          compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
      }
      return false;
  }
```

这里主要是做了前驱的检查状态：
1. 如果前面的`Node` 是`Node.SIGNAL`，根据我们早前看到的这个状态，在这个状态下，其表示当前`node`的后继节点对应的线程需要被唤醒。所以可以放心挂起当前线程。所以返回的是`true`。
2. 如果前驱节点放弃了，也就是状态`>0`, 递归向前到到第一个状态`<=0`的， 然后把自己安排到之后，这之间被遍历到的，就会被`GC`回收掉。
3. 如果没有放弃， 则需要通过`compareAndSetWaitStatus(pred, ws, Node.SIGNAL)`将状态设置为`SIGNAL`。
4. 当`compareAndSetWaitStatus`设置失败则说明`shouldParkAfterFailedAcquire`方法返回`false`，然后会在`acquireQueued()`方法中`for (;;)`死循环中会继续重试，直至`compareAndSetWaitStatus`设置节点状态位为`SIGNAL时shouldParkAfterFailedAcquire`返回`true`时才会执行方法`parkAndCheckInterrupt()`方法。

**shouldParkAfterFailedAcquire(Node, Node)**

如果线程找好安全休息点后，那就可以安心去休息了。此方法就是让线程去休息，真正进入等待状态。
```java
  /**
   * Convenience method to park and then check if interrupted
   *
   * @return {@code true} if interrupted
   */
  private final boolean parkAndCheckInterrupt() {
      LockSupport.park(this);
      return Thread.interrupted();
  }
```
有两种途径可以唤醒该线程：
- 被`unpark()`
- 被`interrupt()`

需要注意的是，`Thread.interrupted()`会清除当前线程的中断标记位。

我们再回到`acquireQueued()`，总结下该函数的具体流程：
> - 结点进入队尾后，检查状态，找到安全休息点；
> - 调用`park()`进入`waiting`状态，等待`unpark()`或`interrupt()`唤醒自己；
> - 被唤醒后，看自己是不是有资格能拿到号。如果拿到，`head`指向当前结点，并返回从入队到拿到号的整个过程中是否被中断过；如果没拿到，继续流程1。

**最后，总结一下acquire()的流程：**

1. 调用自定义同步器的`tryAcquire()`尝试直接去获取资源，如果成功则直接返回；
2. 没成功，则`addWaiter()`将该线程加入等待队列的尾部，并标记为独占模式；
3. `acquireQueued()`使线程在等待队列中休息，有机会时（轮到自己，会被`unpark()`）会去尝试获取资源。获取到资源后才返回。如果在整个等待过程中被中断过，则返回`true`，否则返回`false`。
4. 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断`selfInterrupt()`，将中断补上。

由于此函数是重中之重，我再用流程图总结一下：

![try-acquire][3]

这也就是`ReentrantLock.lock()`的流程，整个函数就是一条`acquire(1)`。

再来个复杂的流程图：
![acquire][6]

#### 3.1.2 释放资源 release(int)

`release(int)`方法是独占模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果彻底释放了（即`state=0`）,它会唤醒等待队列里的其他线程来获取资源。这也正是`unlock()`的语义，当然不仅仅只限于`unlock()`。

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

逻辑并不复杂。它调用`tryRelease()`来释放资源。有一点需要注意的是，它是根据`tryRelease()`的返回值来判断该线程是否已经完成释放掉资源了。所以自定义同步器在设计`tryRelease()`的时候要明确这一点。

##### 3.1.2.1 tryRelease(int)
跟`tryAcquire()`一样，这个方法是需要独占模式的自定义同步器去实现的。正常来说，`tryRelease()`都会成功的，因为这是独占模式，该线程来释放资源，那么它肯定已经拿到独占资源了，直接减掉相应量的资源即可(`state-=arg`)，也不需要考虑线程安全的问题。但要注意它的返回值，上面已经提到了，`release()`是根据`tryRelease()`的返回值来判断该线程是否已经完成释放掉资源了！ 所以自义定同步器在实现时，如果已经彻底释放资源`(state=0)`，要返回`true`，否则返回`false`。

##### 3.1.2.2 unparkSuccessor(Node)

此方法用于唤醒等待队列中下一个线程。

```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}

```

这个函数并不复杂。一句话概括：用`unpark()`唤醒等待队列中最前边的那个未放弃线程，这里我们也用`s`来表示吧。此时，再和`acquireQueued()`联系起来，`s`被唤醒后，进入`if (p == head && tryAcquire(arg))`的判断（即使`p!=head`也没关系，它会再进入`shouldParkAfterFailedAcquire()`寻找一个安全点。这里既然`s`已经是等待队列中最前边的那个未放弃线程了，那么通过`shouldParkAfterFailedAcquire()`的调整，s也必然会跑到`head`的`next`结点，下一次自旋`p==head`就成立啦），然后`s`把自己设置成`head`占用结点，表示自己已经获取到资源了，`acquire()`也返回了。


`release()`是独占模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果彻底释放了（即`state=0`）,它会唤醒等待队列里的其他线程来获取资源。

#### 3.1.3 可中断式获取锁 acquireInterruptibly(int)
我们知道`lock`相较于`synchronized`有一些更方便的特性，比如能响应中断以及超时等待等特性，现在我们依旧采用通过学习源码的方式来看看能够响应中断是怎么实现的。可响应中断式锁可调用方法`lock.lockInterruptibly()`;

而该方法其底层会调用`AQS`的`acquireInterruptibly`方法，源码为：
```java
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

```java
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    //将节点插入到同步队列中
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            //获取锁出队
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                //线程中断抛异常
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

关键信息请看注释，现在看这段代码就很轻松了吧:),与`acquire`方法逻辑几乎一致，唯一的区别是当`parkAndCheckInterrupt`返回`true`时即线程阻塞时该线程被中断，代码抛出被中断异常。

#### 3.1.4 超时等待式获取锁 tryAcquireNanos(int, long)
通过调用`lock.tryLock(timeout,TimeUnit)`方式达到超时等待获取锁的效果，该方法会在三种情况下才会返回：

1. 在超时时间内，当前线程成功获取了锁；
2. 当前线程在超时时间内被中断；
3. 超时时间结束，仍未获得锁返回`false`。

我们仍然通过采取阅读源码的方式来学习底层具体是怎么实现的，该方法会调用`AQS`的方法`tryAcquireNanos()`,源码为：
```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) ||
        //实现超时等待的效果
        doAcquireNanos(arg, nanosTimeout);
}
```

很显然这段源码最终是靠`doAcquireNanos`方法实现超时等待的效果，该方法源码如下：

```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    //1. 根据超时时间和当前时间计算出截止时间
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            //2. 当前线程获得锁出队列
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            // 3.1 重新计算超时时间
            nanosTimeout = deadline - System.nanoTime();
            // 3.2 已经超时返回false
            if (nanosTimeout <= 0L)
                return false;
            // 3.3 线程阻塞等待
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            // 3.4 线程被中断抛出被中断异常
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

![doAcquireNanos][7]

程序逻辑同独占锁可响应中断式获取基本一致，唯一的不同在于获取锁失败后，对超时时间的处理上，在第`1`步会先计算出按照现在时间和超时时间计算出理论上的截止时间，比如当前时间是`8h10min`,超时时间是`10min`，那么根据`deadline = System.nanoTime() + nanosTimeout`计算出刚好达到超时时间时的系统时间就是`8h 10min+10min = 8h 20min`。然后根据`deadline - System.nanoTime()`就可以判断是否已经超时了，比如，当前系统时间是`8h 30min`很明显已经超过了理论上的系统时间`8h 20min`，`deadline - System.nanoTime()`计算出来就是一个负数，自然而然会在`3.2`步中的`If`判断之间返回`false`。如果还没有超时即`3.2`步中的`if`判断为`true`时就会继续执行`3.3`步通过`LockSupport.parkNanos`使得当前线程阻塞，同时在`3.4`步增加了对中断的检测，若检测出被中断直接抛出被中断异常。



### 3.2 共享式 AQS

#### 3.2.1 共享获取资源 acquireShared(int)
此方法是共享模式下线程获取共享资源的顶层入口。它会获取指定量的资源，获取成功则直接返回，获取失败则进入等待队列，直到获取到资源为止，整个过程忽略中断。

```java
/**
 * Acquires in shared mode, ignoring interrupts.  Implemented by
 * first invoking at least once {@link #tryAcquireShared},
 * returning on success.  Otherwise the thread is queued, possibly
 * repeatedly blocking and unblocking, invoking {@link
 * #tryAcquireShared} until success.
 *
 * @param arg the acquire argument.  This value is conveyed to
 *        {@link #tryAcquireShared} but is otherwise uninterpreted
 *        and can represent anything you like.
 */
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

这里`tryAcquireShared()`依然需要自定义同步器去实现。但是`AQS`已经把其返回值的语义定义好了：负值代表获取失败；0代表获取成功，但没有剩余资源；正数表示获取成功，还有剩余资源，其他线程还可以去获取。所以这里`acquireShared()`的流程就是：

1. `tryAcquireShared()`尝试获取资源，成功则直接返回；

2. 失败则通过`doAcquireShared()`进入等待队列，直到获取到资源为止才返回。

#### 3.2.1.1 doAcquireShared(int)

此方法用于将当前线程加入等待队列尾部休息，直到其他线程释放资源唤醒自己，自己成功拿到相应量的资源后才返回。

```java
/**
 * Acquires in shared uninterruptible mode.
 * @param arg the acquire argument
 */
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

跟独占模式比，还有一点需要注意的是，这里只有线程是`head.next`时（“老二”），才会去尝试获取资源，有剩余的话还会唤醒之后的队友。那么问题就来了，假如老大用完后释放了5个资源，而老二需要6个，老三需要1个，老四需要2个。老大先唤醒老二，老二一看资源不够，他是把资源让给老三呢，还是不让？答案是否定的！老二会继续`park()`等待其他线程释放资源，也更不会去唤醒老三和老四了。独占模式，同一时刻只有一个线程去执行，这样做未尝不可；但共享模式下，多个线程是可以同时执行的，现在因为老二的资源需求量大，而把后面量小的老三和老四也都卡住了。当然，这并不是问题，只是`AQS`保证严格按照入队顺序唤醒罢了（保证公平，但降低了并发）。


**setHeadAndPropagate(Node, int)**
```java
/**
 * Sets head of queue, and checks if successor may be waiting
 * in shared mode, if so propagating if either propagate > 0 or
 * PROPAGATE status was set.
 *
 * @param node the node
 * @param propagate the return value from a tryAcquireShared
 */
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus either before
     *     or after setHead) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     */
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```
此方法在`setHead()`的基础上多了一步，就是自己苏醒的同时，如果条件符合（比如还有剩余资源），还会去唤醒后继结点，毕竟是共享模式！

至此，`acquireShared()`也要告一段落了。让我们再梳理一下它的流程：

1. `tryAcquireShared()`尝试获取资源，成功则直接返回；
2. 失败则通过`doAcquireShared()`进入等待队列`park()`，直到被`unpark()/interrupt()`并成功获取到资源才返回。整个等待过程也是忽略中断的。

其实跟`acquire()`的流程大同小异，只不过多了个自己拿到资源后，还会去唤醒后继队友的操作（这才是共享嘛）。


#### 3.2.2 释放共享资源 releaseShared(int)
此方法是共享模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果成功释放且允许唤醒等待线程，它会唤醒等待队列里的其他线程来获取资源。

```java
  /**
   * Releases in shared mode.  Implemented by unblocking one or more
   * threads if {@link #tryReleaseShared} returns true.
   *
   * @param arg the release argument.  This value is conveyed to
   *        {@link #tryReleaseShared} but is otherwise uninterpreted
   *        and can represent anything you like.
   * @return the value returned from {@link #tryReleaseShared}
   */
  public final boolean releaseShared(int arg) {
      if (tryReleaseShared(arg)) {
          doReleaseShared();
          return true;
      }
      return false;
  }
```

此方法的流程也比较简单，一句话：释放掉资源后，唤醒后继。跟独占模式下的`release()`相似，但有一点稍微需要注意：独占模式下的`tryRelease()`在完全释放掉资源（state=0）后，才会返回`true`去唤醒其他线程，这主要是基于独占下可重入的考量；而共享模式下的`releaseShared()`则没有这种要求，共享模式实质就是控制一定量的线程并发执行，那么拥有资源的线程在释放掉部分资源时就可以唤醒后继等待结点。


引用：

[深入理解AbstractQueuedSynchronizer(AQS)][5]





[1]:image/aqs-node-list.png
[2]:image/aqs-node-acquire-state.webp
[3]:image/tryAcquire.webp
[4]:image/acquire-true.webp
[5]:https://www.jianshu.com/p/cc308d82cc71
[6]:image/acquire.webp
[7]:image/doAcquireNanos.webp
