## Semaphore
---

Semaphore翻译成字面意思为 信号量，Semaphore可以控同时访问的线程个数，通过 acquire() 获取一个许可，如果没有就等待，而 release() 释放一个许可。


Semaphore类位于java.util.concurrent包下，它提供了2个构造器:
```
//参数permits表示许可数目，即同时可以允许多少线程进行访问
public Semaphore(int permits) {}
//这个多了一个参数fair表示是否是公平的，即等待时间越久的越先获取许可
public Semaphore(int permits, boolean fair) {}
```
下面说一下Semaphore类中比较重要的几个方法，首先是acquire()、release()方法：
```
public void acquire() throws InterruptedException {  }     //获取一个许可
public void acquire(int permits) throws InterruptedException { }    //获取permits个许可
public void release() { }          //释放一个许可
public void release(int permits) { }    //释放permits个许可
```
acquire()用来获取一个许可，若无许可能够获得，则会一直等待，直到获得许可。

release()用来释放许可。注意，在释放许可之前，必须先获获得许可。

这4个方法都会被阻塞，如果想立即得到执行结果，可以使用下面几个方法：

```
                            //尝试获取一个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
public boolean tryAcquire() { };
                            //尝试获取一个许可，若在指定的时间内获取成功，则立即返回true，否则则立即返回false
public boolean tryAcquire(long timeout, TimeUnit unit) throws InterruptedException { };
                            //尝试获取permits个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
public boolean tryAcquire(int permits) { };
                            //尝试获取permits个许可，若在指定的时间内获取成功，则立即返回true，否则则立即返回false
public boolean tryAcquire(int permits, long timeout, TimeUnit unit) throws InterruptedException { };
```
另外还可以通过availablePermits()方法得到可用的许可数目。



##### 代码：
you can find the code on the [github][1], get the result as you try:
```
public class SemaphoreLearn {

    public static void main(String[] args) {

        CountDownLatch countDownLatch = new CountDownLatch(10);
        Semaphore semaphore = new Semaphore(3);

        for(int i = 0;i<10;i++) {
            new Thread(new SemaphoreThread(countDownLatch,semaphore)).start();
        }

        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("all worker has finish the work");

    }

}

public class SemaphoreThread implements Runnable {

    private CountDownLatch countDownLatch;
    private Semaphore semaphore;

    public SemaphoreThread(CountDownLatch countDownLatch, Semaphore semaphore) {
        this.countDownLatch = countDownLatch;
        this.semaphore = semaphore;
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
    public void run() {
        try {
            semaphore.acquire();
            System.out.println("thread: " +Thread.currentThread().getName()+"is asking a new acquirement.");
            Thread.sleep(1000);
            System.out.println("thread: " +Thread.currentThread().getName()+"release the acquirement.");

        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            countDownLatch.countDown();
            semaphore.release();
        }

    }
}


```


[1]:https://github.com/twentyworld/learn/tree/master/JDKlearn/src/main/java/com/concurrent
