## lock的实现
前文（深入JVM锁机制-synchronized）分析了JVM中的synchronized实现，本文继续分析JVM中的另一种锁Lock的实现。与synchronized不同的是，Lock完全用Java写成，在java这个层面是无关JVM实现的。

在 java.util.concurrent.locks包中有很多Lock的实现类，常用的有ReentrantLock、 ReadWriteLock（实现类ReentrantReadWriteLock），其实现都依赖 java.util.concurrent.AbstractQueuedSynchronizer类，实现思路都大同小异，因此我们以 ReentrantLock作为讲解切入点。

### 1. ReentrantLock的调用过程
经过观察ReentrantLock把所有Lock接口的操作都委派到一个Sync类上，该类继承了AbstractQueuedSynchronizer：
```Java
static abstract class Sync extends AbstractQueuedSynchronizer{

}
```
Sync又有两个子类

```Java
final static class NonfairSync extends Sync{
}
final static class FairSync extends Sync{
}
```

显然是为了支持公平锁和非公平锁而定义，默认情况下为非公平锁。

其实通过上面调用过程及AbstractQueuedSynchronizer的注释可以发现，AbstractQueuedSynchronizer中抽象了绝大多数Lock的功能，而只把tryAcquire方法延迟到子类中实现。 tryAcquire方法的语义在于用具体子类判断请求线程是否可以获得锁，无论成功与否AbstractQueuedSynchronizer都将处理后面的流程。

### 2. 2、锁实现
简单说来，AbstractQueuedSynchronizer会把所有的请求线程构成一个CLH队列，当一个线程执行完毕（lock.unlock()）时会激活自己的后继节点，但正在执行的线程并不在队列中，而那些等待执行的线程全 部处于阻塞状态，经过调查线程的显式阻塞是通过调用LockSupport.park()完成，而LockSupport.park()则调用 sun.misc.Unsafe.park()本地方法，再进一步，HotSpot在Linux中中通过调用pthread_mutex_lock函数把 线程交给系统内核进行阻塞。

与synchronized相同的是，这也是一个虚拟队列，不存在队列实例，仅存在节点之间的前后关系。令人疑惑的是为什么采用CLH队列呢？原生的CLH队列是用于自旋锁，但Doug Lea把其改造为阻塞锁。
当有线程竞争锁时，该线程会首先尝试获得锁，这对于那些已经在队列中排队的线程来说显得不公平，这也是非公平锁的由来，与synchronized实现类似，这样会极大提高吞吐量。 如果已经存在Running线程，则新的竞争线程会被追加到队尾，具体是采用基于CAS的Lock-Free算法，因为线程并发对Tail调用CAS可能会 导致其他线程CAS失败，解决办法是循环CAS直至成功。

AbstractQueuedSynchronizer通过构造一个基于阻塞的CLH队列容纳所有的阻塞线程，而对该队列的操作均通过Lock-Free（CAS）操作，但对已经获得锁的线程而言，ReentrantLock实现了偏向锁的功能。
synchronized 的底层也是一个基于CAS操作的等待队列，但JVM实现的更精细，把等待队列分为ContentionList和EntryList，目的是为了降低线程的出列速度；当然也实现了偏向锁，从数据结构来说二者设计没有本质区别。但synchronized还实现了自旋锁，并针对不同的系统和硬件体系进行了优 化，而Lock则完全依靠系统阻塞挂起等待线程。
当然Lock比synchronized更适合在应用层扩展，可以继承 AbstractQueuedSynchronizer定义各种实现，比如实现读写锁（ReadWriteLock），公平或不公平锁；同时，Lock对 应的Condition也比wait/notify要方便的多、灵活的多。
