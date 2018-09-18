# Thread类认识


Thread类中的实例方法

从Thread类中的实例方法和类方法的角度讲解Thread中的方法，这种区分的角度也有助于理解多线程中的方法。实例方法，只和实例线程（也就是new出来的线程）本身挂钩，和当前运行的是哪个线程无关。


1. start()

start()方法的作用讲得直白点就是通知"线程规划器"，此线程可以运行了，正在等待CPU调用线程对象得run()方法，产生一个异步执行的效果。CPU执行哪个线程的代码具有不确定性。调用start()方法的顺序不代表线程启动的顺序，线程启动顺序具有不确定性。


2、run()

线程开始执行，虚拟机调用的是线程run()方法中的内容。
只有run()没有start()，Thread实例run()方法里面的内容是没有任何异步效果的，全部被main函数执行。换句话说，只有run()而不调用start()启动线程是没有任何意义的。


3、isAlive()

测试线程是否处于活动状态，只要线程启动且没有终止，方法返回的就是true。



4、getId()

在一个Java应用中，有一个long型的全局唯一的线程ID生成器threadSeqNumber，每new出来一个线程都会把这个自增一次，并赋予线程的tid属性，这个是Thread自己做的，用户无法执行一个线程的Id。

5、getName()

我们new一个线程的时候，可以指定该线程的名字，也可以不指定。如果指定，那么线程的名字就是我们自己指定的，getName()返回的也是开发者指定的线程的名字；如果不指定，那么Thread中有一个int型全局唯一的线程初始号生成器threadInitNum，Java先把threadInitNum自增，然后以"Thread-threadInitNum"的方式来命名新生成的线程

6、getPriority()和setPriority(int newPriority)

这两个方法用于获取和设置线程的优先级，优先级高的CPU得到的CPU资源比较多，设置优先级有助于帮"线程规划器"确定下一次选择哪一个线程优先执行。换句话说，两个在等待CPU的线程，优先级高的线程越容易被CPU选择执行。

如果不手动指定，那么线程优先级具有继承性，比如线程A启动线程B，那么线程B的优先级和线程A的优先级相同

CPU会尽量将执行资源让给优先级比较高的线程。



7、isDaeMon、setDaemon(boolean on)

讲解两个方法前，首先要知道理解一个概念。Java中有两种线程，一种是用户线程，一种是守护线程。守护线程是一种特殊的线程，它的作用是为其他线程的运行提供便利的服务，最典型的应用便是GC线程。如果进程中不存在非守护线程了，那么守护线程自动销毁，因为没有存在的必要，为别人服务，结果服务的对象都没了，当然就销毁了。理解了这个概念后，看一下例子：

```Java
public class MyThread11 extends Thread {
    private int i = 0;

    public void run() {
        try {
            while (true) {
                i++;
                System.out.println("i = " + i);
                Thread.sleep(1000);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

public static void main(String[] args) {
    try {
        MyThread11 mt = new MyThread11();
        mt.setDaemon(true);
        mt.start();
        Thread.sleep(5000);
        System.out.println("我离开thread对象再也不打印了，我停止了！");
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

我们将MyThread11线程设置为守护线程，看到第6行的那句话，而i停在6不会再运行了。这说明，main线程运行了5秒多结束，而i每隔1秒累加一次，5秒后main线程执行完结束了，MyThread11作为守护线程，main函数都运行完了，自然也没有存在的必要了，就自动销毁了，因此也就没有再往下打印数字。

关于守护线程，有一个细节注意下，setDaemon(true)必须在线程start()之前

8、interrupt()

这是一个有点误导性的名字，实际上Thread类的interrupt()方法无法中断线程。
interrupt()方法的作用实际上是：在线程受到阻塞时抛出一个中断信号，这样线程就得以退出阻塞状态。换句话说，没有被阻塞的线程，调用interrupt()方法是不起作用的。

9、isInterrupted()

测试线程是否已经中断，但不清除状态标识。
