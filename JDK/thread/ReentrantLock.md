ReentrantLock
---

ReentrantLock，意思是“可重入锁”，关于可重入锁的概念在下一节讲述。ReentrantLock是唯一实现了Lock接口的类，并且ReentrantLock提供了更多的方法。下面通过一些实例看具体看一下如何使用ReentrantLock。

### 1. Lock代码实例：
你可以在[github][1]中看到。
#### 1.1 lock();
首先，我们尝试写一个错误的代码：
```Java
public class ReentrantLockWrongSample {
    public static void main(String[] args) {

        ArrayList<Integer> arrayList = new ArrayList<Integer>();

        WrongSampleThread thread1 = new WrongSampleThread(arrayList);
        WrongSampleThread thread2 = new WrongSampleThread(arrayList);

        new Thread(thread1).start();
        new Thread(thread1).start();
        new Thread(thread2).start();
        new Thread(thread2).start();

    }
}

public class WrongSampleThread implements Runnable {
    private ArrayList<Integer> arrayList;
    public WrongSampleThread(ArrayList<Integer> arrayList) {
        this.arrayList = arrayList;
    }

    @Override
    public void run() {
        Lock lock = new ReentrantLock();    //注意这个地方
        lock.lock();
        try {
            System.out.println(Thread.currentThread() + "得到了锁");
            for (int i = 0; i < 5; i++) arrayList.add(i);
            Thread.sleep(100);
        } catch (Exception e) {
        } finally {
            System.out.println(Thread.currentThread() + "释放了锁");
            lock.unlock();
        }
    }
}
```
output:
```Java
Thread[Thread-0,5,main]得到了锁
Thread[Thread-3,5,main]得到了锁
Thread[Thread-2,5,main]得到了锁
Thread[Thread-1,5,main]得到了锁
Thread[Thread-1,5,main]释放了锁
Thread[Thread-0,5,main]释放了锁
Thread[Thread-3,5,main]释放了锁
Thread[Thread-2,5,main]释放了锁
```
第二个线程怎么会在第一个线程释放锁之前得到了锁？原因在于，在insert方法中的lock变量是局部变量，每个线程执行该方法时都会保存一个副本，那么理所当然每个线程执行到lock.lock()处获取的是不同的锁，所以就不会发生冲突。
所以我们只需要把Lock设置为类的属性：
```Java
public class ReentrantLockRightSample {
    public static void main(String[] args) {
        ArrayList<Integer> list = new ArrayList<Integer>();
        Lock lock = new ReentrantLock();    //注意这个地方

        RightSampleThread thread1 = new RightSampleThread(lock, list);
        RightSampleThread thread2 = new RightSampleThread(lock, list);

        new Thread(thread1).start();
        new Thread(thread2).start();
    }
}

public class RightSampleThread implements Runnable {
    private Lock lock;
    private List<Integer> list;

    public RightSampleThread(Lock lock, List<Integer> list) {
        this.lock = lock;
        this.list = list;
    }

    @Override
    public void run() {
        lock.lock();
        try {
            System.out.println(Thread.currentThread() + "得到了锁");
            for (int i = 0; i < 5; i++) list.add(i);
            Thread.sleep(200);
        } catch (Exception e) {
            // TODO: handle exception
        } finally {
            System.out.println(Thread.currentThread() + "释放了锁");
            lock.unlock();
        }
    }
}
```
output:
```
Thread[Thread-0,5,main]得到了锁
Thread[Thread-0,5,main]释放了锁
Thread[Thread-1,5,main]得到了锁
Thread[Thread-1,5,main]释放了锁
```

#### 3.2 tryLock()

```Java
public class ReentrantLockTryLockLearn {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<Integer>();
        Lock lock = new ReentrantLock();    //注意这个地方

        TryLockThread thread1 = new TryLockThread(list, lock);
        TryLockThread thread2 = new TryLockThread(list, lock);
        TryLockWithTimeThread thread3 = new TryLockWithTimeThread(list, lock);
        new Thread(thread1).start();
        new Thread(thread2).start();
        new Thread(thread3).start();
    }
}

public class TryLockThread implements Runnable {
    private List<Integer> list;
    private Lock lock;

    public TryLockThread(List<Integer> list, Lock lock) {
        this.list = list;
        this.lock = lock;
    }
    @Override
    public void run() {
        if (lock.tryLock()) {
            try {
                System.out.println(Thread.currentThread() + "得到了锁");
                for (int i = 0; i < 10; i++) {
                    list.add(i);
                }
                Thread.sleep(500);
            } catch (Exception e) {
                // TODO: handle exception
            } finally {
                System.out.println(Thread.currentThread() + "释放了锁");
                lock.unlock();
            }
        } else {
            System.out.println(Thread.currentThread() + "获取锁失败");
        }
    }
}

public class TryLockWithTimeThread implements Runnable {
    private List<Integer> list;
    private Lock lock;

    public TryLockWithTimeThread(List<Integer> list, Lock lock) {
        this.list = list;
        this.lock = lock;
    }
    @Override
    public void run() {
        try {
           if (lock.tryLock(5, TimeUnit.SECONDS)) {
                try {
                    System.out.println(Thread.currentThread() + "得到了锁");
                    for (int i = 0; i < 10; i++) {
                        list.add(i);
                    }
                    Thread.sleep(100);
                } catch (Exception e) {
                    // TODO: handle exception
                } finally {
                    System.out.println(Thread.currentThread() + "释放了锁");
                    lock.unlock();
                }
            } else {
                System.out.println(Thread.currentThread() + "获取锁失败");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
output:
```
Thread[Thread-0,5,main]得到了锁
Thread[Thread-1,5,main]获取锁失败
Thread[Thread-0,5,main]释放了锁
Thread[Thread-2,5,main]得到了锁
Thread[Thread-2,5,main]释放了锁
```
#### 3.3 lockInterruptibly()
```Java
public class LockInterruptiblyLockLearn {
    public static void main(String[] args) {
        Lock lock = new ReentrantLock();
        LockInterruptiblyThread interruptiblyThread1 = new LockInterruptiblyThread(lock);
        LockInterruptiblyThread interruptiblyThread2 = new LockInterruptiblyThread(lock);
        Thread thread1 = new Thread(interruptiblyThread1);
        Thread thread2 = new Thread(interruptiblyThread2);
        thread1.start();
        thread2.start();
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        thread2.interrupt();
    }
}

public class LockInterruptiblyThread implements Runnable {
    private Lock lock;
    public LockInterruptiblyThread(Lock lock) {
        this.lock = lock;
    }
    @Override
    public void run() {
        try {
            //注意，如果需要正确中断等待锁的线程，必须将获取锁放在外面，然后将InterruptedException抛出
            lock.lockInterruptibly();
            try {
                System.out.println(Thread.currentThread() + "得到了锁");
                long startTime = System.currentTimeMillis();
                for (; ; ) {
                    if (System.currentTimeMillis() - startTime >= 5000) break;
                    //插入数据
                }
            } finally {
                System.out.println(Thread.currentThread().getName() + "执行finally");
                lock.unlock();
                System.out.println(Thread.currentThread() + "释放了锁");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
            System.out.println(Thread.currentThread().getName() + "被中断");
        }
    }
}
```
output:
```Java
Thread[Thread-0,5,main]得到了锁
java.lang.InterruptedException
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchronizer.java:898)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1222)
	at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
	at com.lock.reentrantLock.reentrantLockApi.thread.LockInterruptiblyThread.run(LockInterruptiblyThread.java:38)
	at java.lang.Thread.run(Thread.java:748)
Thread-1被中断
Thread-0执行finally
Thread[Thread-0,5,main]释放了锁
```
### 2. Condition 例子：
Condition 将 Object 监视器方法（wait、notify 和 notifyAll）分解成截然不同的对象，以便通过将这些对象与任意 Lock 实现组合使用，为每个对象提供多个等待 set （wait-set）。

其中，Lock 替代了 synchronized 方法和语句的使用，Condition 替代了 Object 监视器方法的使用。

在Condition中，用await()替换wait()，用signal()替换notify()，用signalAll()替换notifyAll()，传统线程的通信方式，Condition都可以实现，这里注意，Condition是被绑定到Lock上的，要创建一个Lock的Condition必须用newCondition()方法。

这样看来，Condition和传统的线程通信没什么区别，Condition的强大之处在于它可以为多个线程间建立不同的Condition。

你可以在[github][4]中发现源码。


首先我们建立一个`ArrayBuffer`类，这个类中，我们设置了两个方法，`read()`和`put(String number)`。
`write`方法是向队列中写入数据，`read`方法是从队列中读取数据。
```Java
public class ArrayBuffer {

    private Lock lock = new ReentrantLock();
    private Condition write;
    private Condition read;
    public ArrayBuffer() {
        write = lock.newCondition();
        read = lock.newCondition();
    }
    private String[] array = new String[30];
    private int putPoint, takePoint;
    private int count = 0;

    public String read() {
        lock.lock();
        try {

            if (count == 0) {
                try {
                    System.out.println("buffer is empty");
                    write.signal();
                    read.await();
                } catch (InterruptedException e) {e.printStackTrace();}
            }
            System.out.println("takePoint: " + takePoint);
            String temp = array[takePoint()];
            count--;
            takePoint++;
            return temp;
        }finally {
            lock.unlock();
        }
    }

    public void put(String number) {
        lock.lock();
        try{
            if (count>=array.length){
                try {
                    System.out.println("buffer is full.");
                    read.signal();
                    write.await();
                    System.out.println("count: " + count);
                } catch (InterruptedException e) {e.printStackTrace();}
            }
            array[putPoint()] = number;
            count++;
            System.out.println("putPoint: " + putPoint);
            putPoint++;
        }finally {lock.unlock();}
    }
    private int putPoint() {
        return putPoint%(array.length);
    }

    private int takePoint() {
        return takePoint%(array.length);
    }

}

```
其次，我们建立两个不同的thread，分别读取和写入。
```Java
public class ReadThread implements Runnable {
    private ArrayBuffer buffer;
    public ReadThread(ArrayBuffer buffer) {
        this.buffer = buffer;
    }
    @Override
    public void run() {
        System.out.println("read running.");
        for (int i =0;i<500;i++) {
            System.out.println("read result: " + buffer.read());
        }
    }
}

public class WriteThread implements Runnable {
    private ArrayBuffer buffer;
    public WriteThread(ArrayBuffer buffer) {
        this.buffer = buffer;
    }
    @Override
    public void run() {
        System.out.println("write running");
        for (int i =0;i<500;i++) {
            buffer.put(i+"");
        }
    }
}
```
测试类
```Java
public class ConditionOnReadWriteLearn {
    public static void main(String[] args) {
        ArrayBuffer buffer = new ArrayBuffer();
        ReadThread read = new ReadThread(buffer);
        WriteThread write = new WriteThread(buffer);

        new Thread(write).start();
        new Thread(read).start();

    }
}
```
output:
```
read running.
write running
putPoint: 0
takePoint: 0
read result: 0
putPoint: 1
···
putPoint: 9
putPoint: 10
takePoint: 1
read result: 1
putPoint: 11
···
putPoint: 31
buffer is full.
takePoint: 2
read result: 2
···
takePoint: 31
read result: 31
buffer is empty
count: 0
putPoint: 32
putPoint: 33
putPoint: 34
···
```

### 3. Java 同步器 AbstractQueuedSynchronizer


---
引用文章：

[轻松学习java可重入锁(ReentrantLock)的实现原理][2]

[Java并发控制：ReentrantLock Condition使用详解][3]

[深入浅出java同步器AQS][5]

[AbstractQueuedSynchronizer源码剖析（六）- 深刻解析与模拟线程竞争资源][6]

[再谈 AbstractQueuedSynchronizer 2 ：共享模式与基于 Condition 的等待 / 通知机制实现][7]

[AbstractQueuedSynchronizer的介绍和原理分析][8]

[再谈 AbstractQueuedSynchronizer 1 ：独占模式][9]

[AQS(AbstractQueuedSynchronizer)源代码分析][11]

[AbstractQueuedSynchronizer源码解读][10]
---
[1]:https://github.com/twentyworld/learn/tree/master/JDKlearn/src/main/java/com/lock/reentrantLock
[2]:https://blog.csdn.net/yanyan19880509/article/details/52345422
[3]:https://www.cnblogs.com/hongdada/p/6150699.html
[4]:
[5]:https://www.jianshu.com/p/d8eeb31bee5c
[6]:https://blog.csdn.net/pfnie/article/details/53191892
[7]:http://www.importnew.com/26300.html
[8]:http://ifeve.com/introduce-abstractqueuedsynchronizer/
[9]:http://www.importnew.com/26284.html
[10]:https://www.cnblogs.com/micrari/p/6937995.html
[11]:https://blog.csdn.net/qq_36372507/article/details/79308182
