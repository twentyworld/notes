## CyclicBarrier

---

CyclicBarrier 的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活。

CyclicBarrier默认的构造方法是CyclicBarrier(int parties)，其参数表示屏障拦截的线程数量，每个线程调用await方法告诉CyclicBarrier我已经到达了屏障，然后当前线程被阻塞。CyclicBarrier还提供一个更高级的构造函数CyclicBarrier(int parties, Runnable barrierAction)，用于在线程到达屏障时，优先执行barrierAction这个Runnable对象，方便处理更复杂的业务场景。线程进入屏障通过CyclicBarrier的await()方法。

##### await()
```
public int await() throws InterruptedException, BrokenBarrierException {};
public int await(long timeout, TimeUnit unit)throws InterruptedException,BrokenBarrierException,TimeoutException {};
```
await()比较常用，用来挂起当前线程，直至所有线程都到达barrier状态再同时执行后续任务；

await(long timeout, TimeUnit unit)是让这些线程等待至一定的时间，如果还有线程没有到达barrier状态就直接让到达barrier的线程执行后续任务。


##### 代码实例：
you can find the code on the [github][1], get the result as you try:
```
public class CyclicBarrierLearn {
    public static void main(String[] args) throws InterruptedException {
//      CyclicBarrier cyclicBarrier = new CyclicBarrier(100);

        CyclicBarrier cyclicBarrier = new CyclicBarrier(10, new Runnable() {
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName());
            }
        });

        for(int i =0;i<9;i++) {
            new Thread(new CyclicBarrierThread(cyclicBarrier)).start();
        }

        Thread.sleep(4000);

        new Thread(new CyclicBarrierThread(cyclicBarrier)).start();

        System.out.println("all invoked.");
    }


}


class CyclicBarrierThread implements Runnable {

    private CyclicBarrier cyclicBarrier;

    public CyclicBarrierThread(CyclicBarrier cyclicBarrier) {
        this.cyclicBarrier = cyclicBarrier;
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

        System.out.println("thread "+Thread.currentThread().getName()+" is writing data.");

        try {
            Thread.sleep(1000);
            System.out.println("writing end.");
//            cyclicBarrier.await(3000, TimeUnit.MILLISECONDS);
            cyclicBarrier.await();
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }

        System.out.println("all sub thread off.");
    }
}

```

[1]:https://github.com/twentyworld/learn/tree/master/JDKlearn/src/main/java/com/concurrent
