## CountDownLatch

---

CountDownLatch是通过一个计数器来实现的，计数器的初始值为线程的数量。每当一个线程完成了自己的任务后，计数器的值就会减1。当计数器值到达0时，它表示所有的线程已经完成了任务，然后在闭锁上等待的线程就可以恢复执行任务。


```
//Main thread start
//Create CountDownLatch for N threads
//Create and start N threads
//Main thread wait on latch
//N threads completes there tasks are returns
//Main thread resume execution
```

构造器中的计数值（count）实际上就是闭锁需要等待的线程数量。这个值只能被设置一次，而且CountDownLatch没有提供任何机制去重新设置这个计数值。

与CountDownLatch的第一次交互是主线程等待其他线程。主线程必须在启动其他线程后立即调用CountDownLatch.await()方法。这样主线程的操作就会在这个方法上阻塞，直到其他线程完成各自的任务。

其他N 个线程必须引用闭锁对象，因为他们需要通知CountDownLatch对象，他们已经完成了各自的任务。这种通知机制是通过 CountDownLatch.countDown()方法来完成的；每调用一次这个方法，在构造函数中初始化的count值就减1。所以当N个线程都调 用了这个方法，count的值等于0，然后主线程就能通过await()方法，恢复执行自己的任务。

##### 举例作用：

1. 实现最大的并行性：有时我们想同时启动多个线程，实现最大程度的并行性。例如，我们想测试一个单例类。如果我们创建一个初始计数为1的CountDownLatch，并让所有线程都在这个锁上等待，那么我们可以很轻松地完成测试。我们只需调用 一次countDown()方法就可以让所有的等待线程同时恢复执行。
2. 开始执行前等待n个线程完成各自任务：例如应用程序启动类要确保在处理用户请求前，所有N个外部系统已经启动和运行了。
3. 死锁检测：一个非常方便的使用场景是，你可以使用n个线程访问共享资源，在每次测试阶段的线程数目是不同的，并尝试产生死锁。


面试准备以下一些CountDownLatch相关的问题：

1. 解释一下CountDownLatch概念?
2. CountDownLatch 和CyclicBarrier的不同之处?
3. 给出一些CountDownLatch使用的例子?
4. CountDownLatch 类中主要的方法?

但是一般情况下, 我们会联系其他类共同探讨。

##### 代码：
you can find the code on the [github][1], get the result as you try:
```Java
public class CountDownLatchLearn {
    public static void main(String[] args) {
        CountDownLatch countDownLatch = new CountDownLatch(10);
        for (int i = 0; i < 10; i++) {
            new Thread(new CountDownLatchThread(countDownLatch)).start();
        }

        System.out.println("waiting two threads running off.");
        try {
            countDownLatch.await();
            System.out.println("main thread running.");
            System.out.println("end.");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }


    }
}

// you can create a new class named CountDownLatchThread.
class CountDownLatchThread implements Runnable {

    private CountDownLatch countDownLatch;

    public CountDownLatchThread(CountDownLatch countDownLatch) {
        this.countDownLatch = countDownLatch;
    }

    /**
     * When an object implementing interface <code>Runnable</code> is used
     * to create a thread, starting the thread causes the object's
     * <code>run</code> method to be called in that separately executing
     * thread.
     * <p>
     * The general contract of the method <code>run</code> is that it may
     * take any action whatsoever.
     *
     * @see Thread#run()
     */
    @Override
    public void run( ) {

        System.out.println("线程："+ Thread.currentThread().getName()+" is running.");
        try {
            Thread.sleep(1000);
            System.out.println("线程："+ Thread.currentThread().getName()+" finished.");

        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            countDownLatch.countDown();
        }

    }
}

```


[1]:https://github.com/twentyworld/learn/tree/master/JDKlearn/src/main/java/com/concurrent
