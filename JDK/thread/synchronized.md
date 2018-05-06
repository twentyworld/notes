## synchronized
---
### 1. 进程与线程
感谢[编程思想之多线程与多进程(1)——以操作系统的角度述说线程与进程][1]
#### 1.1 任务调度
大部分操作系统(如Windows、Linux)的任务调度是采用时间片轮转的抢占式调度方式，也就是说一个任务执行一小段时间后强制暂停去执行下一个任务，每个任务轮流执行。

任务执行的一小段时间叫做时间片，任务正在执行时的状态叫运行状态，任务执行一段时间后强制暂停去执行下一个任务，被暂停的任务就处于就绪状态等待下一个属于它的时间片的到来。这样每个任务都能得到执行，由于CPU的执行效率非常高，时间片非常短，在各个任务之间快速地切换，给人的感觉就是多个任务在“同时进行”，这也就是我们所说的并发(别觉得并发有多高深，它的实现很复杂，但它的概念很简单，就是一句话：多个任务同时执行)。

#### 1.2 进程
我们都知道计算机的核心是CPU，它承担了所有的计算任务；而操作系统是计算机的管理者，它负责任务的调度、资源的分配和管理，统领整个计算机硬件；应用程序侧是具有某种功能的程序，程序是运行于操作系统之上的。

进程是一个具有一定独立功能的程序在一个数据集上的一次动态执行的过程，是操作系统进行资源分配和调度的一个独立单位，是应用程序运行的载体。进程是一种抽象的概念，从来没有统一的标准定义。进程一般由程序、数据集合和进程控制块三部分组成。程序用于描述进程要完成的功能，是控制进程执行的指令集；数据集合是程序在执行时所需要的数据和工作区；程序控制块(Program Control Block，简称PCB)，包含进程的描述信息和控制信息，是进程存在的唯一标志。

进程具有的特征：
1. 动态性：进程是程序的一次执行过程，是临时的，有生命期的，是动态产生，动态消亡的；
2. 并发性：任何进程都可以同其他进程一起并发执行；
3. 独立性：进程是系统进行资源分配和调度的一个独立单位；
4. 结构性：进程由程序、数据和进程控制块三部分组成。

####1.3 线程
在早期的操作系统中并没有线程的概念，进程是能拥有资源和独立运行的最小单位，也是程序执行的最小单位。任务调度采用的是时间片轮转的抢占式调度方式，而进程是任务调度的最小单位，每个进程有各自独立的一块内存，使得各个进程之间内存地址相互隔离。

后来，随着计算机的发展，对CPU的要求越来越高，进程之间的切换开销较大，已经无法满足越来越复杂的程序的要求了。于是就发明了线程，线程是程序执行中一个单一的顺序控制流程，是程序执行流的最小单元，是处理器调度和分派的基本单位。一个进程可以有一个或多个线程，各个线程之间共享程序的内存空间(也就是所在进程的内存空间)。一个标准的线程由线程ID、当前指令指针(PC)、寄存器和堆栈组成。而进程由内存空间(代码、数据、进程空间、打开的文件)和一个或多个线程组成。

#### 1.4 多核与多线程
多核(心)处理器是指在一个处理器上集成多个运算核心从而提高计算能力，也就是有多个真正并行计算的处理核心，每一个处理核心对应一个内核线程。内核线程（Kernel Thread， KLT）就是直接由操作系统内核支持的线程，这种线程由内核来完成线程切换，内核通过操作调度器对线程进行调度，并负责将线程的任务映射到各个处理器上。一般一个处理核心对应一个内核线程，比如单核处理器对应一个内核线程，双核处理器对应两个内核线程，四核处理器对应四个内核线程。

现在的电脑一般是双核四线程、四核八线程，是采用超线程技术将一个物理处理核心模拟成两个逻辑处理核心，对应两个内核线程，所以在操作系统中看到的CPU数量是实际物理CPU数量的两倍。

超线程技术就是利用特殊的硬件指令，把一个物理芯片模拟成两个逻辑处理核心，让单个处理器都能使用线程级并行计算，进而兼容多线程操作系统和软件，减少了CPU的闲置时间，提高的CPU的运行效率。这种超线程技术(如双核四线程)由处理器硬件的决定，同时也需要操作系统的支持才能在计算机中表现出来。

程序一般不会直接去使用内核线程，而是去使用内核线程的一种高级接口——轻量级进程（Light Weight Process，LWP），轻量级进程就是我们通常意义上所讲的线程(我们在这称它为用户线程)，由于每个轻量级进程都由一个内核线程支持，因此只有先支持内核线程，才能有轻量级进程。用户线程与内核线程的对应关系有三种模型：一对一模型、多对一模型、多对多模型，在这以4个内核线程、3个用户线程为例对三种模型进行说明。

##### 1.4.1 一对一模型
对于一对一模型来说，一个用户线程就唯一地对应一个内核线程(反过来不一定成立，一个内核线程不一定有对应的用户线程)。这样，如果CPU没有采用超线程技术(如四核四线程的计算机)，一个用户线程就唯一地映射到一个物理CPU的线程，线程之间的并发是真正的并发。一对一模型使用户线程具有与内核线程一样的优点，一个线程因某种原因阻塞时其他线程的执行不受影响；此处，一对一模型也可以让多线程程序在多处理器的系统上有更好的表现。

但一对一模型也有两个缺点：1.许多操作系统限制了内核线程的数量，因此一对一模型会使用户线程的数量受到限制；2.许多操作系统内核线程调度时，上下文切换的开销较大，导致用户线程的执行效率下降。

##### 1.4.2 多对一模型
多对一模型将多个用户线程映射到一个内核线程上，线程之间的切换由用户态的代码来进行，因此相对一对一模型，多对一模型的线程切换速度要快许多；此外，多对一模型对用户线程的数量几乎无限制。但多对一模型也有两个缺点：1.如果其中一个用户线程阻塞，那么其它所有线程都将无法执行，因为此时内核线程也随之阻塞了；2.在多处理器系统上，处理器数量的增加对多对一模型的线程性能不会有明显的增加，因为所有的用户线程都映射到一个处理器上了。

##### 1.4.3 多对多模型

多对多模型结合了一对一模型和多对一模型的优点，将多个用户线程映射到多个内核线程上。多对多模型的优点有：1.一个用户线程的阻塞不会导致所有线程的阻塞，因为此时还有别的内核线程被调度来执行；2.多对多模型对用户线程的数量没有限制；3.在多处理器的操作系统中，多对多模型的线程也能得到一定的性能提升，但提升的幅度不如一对一模型的高。

在现在流行的操作系统中，大都采用多对多的模型。

![多对多模型][2]
#### 1.5 线程的生命周期
当线程的数量小于处理器的数量时，线程的并发是真正的并发，不同的线程运行在不同的处理器上。但当线程的数量大于处理器的数量时，线程的并发会受到一些阻碍，此时并不是真正的并发，因为此时至少有一个处理器会运行多个线程。

在单个处理器运行多个线程时，并发是一种模拟出来的状态。操作系统采用时间片轮转的方式轮流执行每一个线程。现在，几乎所有的现代操作系统采用的都是时间片轮转的抢占式调度方式，如我们熟悉的Unix、Linux、Windows及Mac OS X等流行的操作系统。

我们知道线程是程序执行的最小单位，也是任务执行的最小单位。在早期只有进程的操作系统中，进程有五种状态，创建、就绪、运行、阻塞(等待)、退出。早期的进程相当于现在的只有单个线程的进程，那么现在的多线程也有五种状态，现在的多线程的生命周期与早期进程的生命周期类似。

#### 1.6 线程优先级与线程安全
##### 1.6.1 线程优先级
现在主流操作系统(如Windows、Linux、Mac OS X)的任务调度除了具有前面提到的时间片轮转的特点外，还有优先级调度(Priority Schedule)的特点。优先级调度决定了线程按照什么顺序轮流执行，在具有优先级调度的系统中，线程拥有各自的线程优先级(Thread Priority)。具有高优先级的线程会更早地执行，而低优先级的线程通常要等没有更高优先级的可执行线程时才会被执行。

线程的优先级可以由用户手动设置，此外系统也会根据不同情形调整优先级。通常情况下，频繁地进入等待状态(进入等待状态会放弃之前仍可占用的时间份额)的线程(如IO线程)，比频繁进行大量计算以至于每次都把所有时间片全部用尽的线程更受操作系统的欢迎。因为频繁进入等待的线程只会占用很少的时间，这样操作系统可以处理更多的任务。我们把频繁等待的线程称之为IO密集型线程(IO Bound Thread)，而把很少等待的线程称之为CPU密集型线程(CPU Bound Thread)。IO密集型线程总是比CPU密集型线程更容易得到优先级的提升。
##### 1.6.2 线程饿死:
在优先级调度下，容易出现一种线程饿死的现象。一个线程饿死是说它的优先级较低，在它执行之前总是有比它优先级更高的线程等待执行，因此这个低优先级的线程始终得不到执行。当CPU密集型的线程优先级较高时，其它低优先级的线程就很可能出现饿死的情况；当IO密集型线程优先级较高时，其它线程相对不容易造成饿死的善，因为IO线程有大量的等待时间。为了避免线程饿死，调度系统通常会逐步提升那些等待了很久而得不到执行的线程的优先级。这样，一个线程只要它等待了足够长的时间，其优先级总会被提升到可以让它执行的程度，也就是说这种情况下线程始终会得到执行，只是时间的问题。

在优先级调度环境下，线程优先级的改变有三种方式：
1. 用户指定优先级；
2. 根据进入等待状态的频繁程度提升或降低优先级(由操作系统完成)；
3. 长时间得不到执行而被提升优先级。

##### 1.6.3 线程安全与锁

所谓同步(synchronization)就是指一个线程访问数据时，其它线程不得对同一个数据进行访问，即同一时刻只能有一个线程访问该数据，当这一线程访问结束时其它线程才能对这它进行访问。同步最常见的方式就是使用锁(Lock)，也称为线程锁。锁是一种非强制机制，每一个线程在访问数据或资源之前，首先试图获取(Acquire)锁，并在访问结束之后释放(Release)锁。在锁被占用时试图获取锁，线程会进入等待状态，直到锁被释放再次变为可用。

###### 二元信号量
二元信号量(Binary Semaphore)是一种最简单的锁，它有两种状态：占用和非占用。它适合只能被唯一一个线程独占访问的资源。当二元信号量处于非占用状态时，第一个试图获取该二元信号量锁的线程会获得该锁，并将二元信号量锁置为占用状态，之后其它试图获取该二元信号量的线程会进入等待状态，直到该锁被释放。

###### 信号量
多元信号量允许多个线程访问同一个资源，多元信号量简称信号量(Semaphore)，对于允许多个线程并发访问的资源，这是一个很好的选择。一个初始值为N的信号量允许N个线程并发访问。线程访问资源时首先获取信号量锁，进行如下操作：
1. 将信号量的值减1；
2. 如果信号量的值小于0，则进入等待状态，否则继续执行；
访问资源结束之后，线程释放信号量锁，进行如下操作：
1. 将信号量的值加1；
2. 如果信号量的值小于1(等于0)，唤醒一个等待中的线程；

###### 互斥量
互斥量(Mutex)和二元信号量类似，资源仅允许一个线程访问。与二元信号量不同的是，信号量在整个系统中可以被任意线程获取和释放，也就是说，同一个信号量可以由一个线程获取而由另一线程释放。而互斥量则要求哪个线程获取了该互斥量锁就由哪个线程释放，其它线程越俎代庖释放互斥量是无效的。

###### 临界区
临界区(Critical Section)是一种比互斥量更加严格的同步手段。互斥量和信号量在系统的任何进程都是可见的，也就是说一个进程创建了一个互斥量或信号量，另一进程试图获取该锁是合法的。而临界区的作用范围仅限于本进程，其它的进程无法获取该锁。除此之处，临界区与互斥量的性质相同。
###### 读写锁
读写锁(Read-Write Lock)允许多个线程同时对同一个数据进行读操作，而只允许一个线程进行写操作。这是因为读操作不会改变数据的内容，是安全的；而写操作会改变数据的内容，是不安全的。对同一个读写锁，有两种获取方式：共享的(Shared)和独占的(Exclusive)。当锁处于自由状态时，试图以任何一种方式获取锁都能成功，并将锁置为对应的状态；如果锁处于共享状态，其它线程以共享方式获取该锁，仍然能成功，此时该锁分配给了多个线程；如果其它线程试图如独占的方式获取处于共享状态的锁，它必须等待所有线程释放该锁；处于独占状态的锁阻止任何线程获取该锁，不论它们以何种方式。
### 2 Java 中的线程同步
#### 2.1 单线程
任何程序至少有一个线程，即使你没有主动地创建线程，程序从一开始执行就有一个默认的线程，被称为主线程，只有一个线程的程序称为单线程程序。如下面这一简单的代码，没有显示地创建一个线程，程序从main开始执行，main本身就是一个线程(主线程)，单个线程从头执行到尾。

#### 2.2 多线程
大部分情况下选择实现Runnable接口的方式会优于继承Thread的方式，因为：
1. 从 Thread 类继承会强加类层次；
2. 有些类不能继承Thread类，如要作为线程运行的类已经是某一个类的子类了，但Java只支持单继承，所以不能再继承Thread类了。

#### 2.3 线程控制
##### 2.3.1 线程等待(wait、notify、notifyAll)
在Java的Object类中有三个final的方法允许线程之间进行资源对象锁的通信，他们分别是： wait(), notify() and notifyAll()。
###### wait
Object的wait方法有三个重载方法，其中一个方法wait() 是无限期(一直)等待，直到其它线程调用notify或notifyAll方法唤醒当前的线程；另外两个方法wait(long timeout) 和wait(long timeout, int nanos)允许传入 当前线程在被唤醒之前需要等待的时间，timeout为毫秒数，nanos为纳秒数。

###### notify
notify方法只唤醒一个等待（对象的）线程并使该线程开始执行。所以如果有多个线程等待一个对象，这个方法只会唤醒其中一个线程，选择哪个线程取决于操作系统对多线程管理的实现。

###### notifyAll
notifyAll 会唤醒所有等待(对象的)线程，尽管哪一个线程将会第一个处理取决于操作系统的实现。

这些方法可以使用于“生产者-消费者”问题，消费者是在队列中等待对象的线程，生产者是在队列中释放对象并通知其他线程的线程。

你可以在[github][5]上看到源码：

一个java bean类，线程将会使用它并调用wait和notify方法。
```java
public class Message {
    private String msg;
    public Message(String str){this.msg=str;}
    public String getMsg() {return msg;}
    public void setMsg(String str) {this.msg=str;}
}
```
一个Waiter类，等待其它的线程调用notify方法以唤醒线程完成处理。注意等待线程必须通过加synchronized同步锁拥有Message对象的监视器。
```java
public class Waiter implements Runnable {
    private Message msg;
    public Waiter(Message m) {this.msg = m;}
    @Override
    public void run() {
        String name = Thread.currentThread().getName();
        synchronized (msg) {
            try {
                System.out.println(name + " waiting to get notified at time:" + System.currentTimeMillis());
                msg.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(name + " waiter thread got notified at time:" + System.currentTimeMillis());
            //process the message now
            System.out.println(name + " processed: " + msg.getMsg());
        }
    }
}
```
一个Notifier类，处理Message对象并调用notify方法唤醒等待Message对象的线程。注意synchronized代码块被用于持有Message对象的监视器。
```java
public class Notifier implements Runnable {
    private Message msg;
    public Notifier(Message msg) {this.msg = msg;}
    @Override
    public void run() {
        String name = Thread.currentThread().getName();
        System.out.println(name + " started");
        try {
            Thread.sleep(1000);
            synchronized (msg) {
                msg.setMsg(name + " Notifier work done");
                msg.notify();
                // msg.notifyAll();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
一个测试类，交付创建多个等待线程和一个通过线程，并启动这些线程。
```java
public class WaitNotifyTest {
    public static void main(String[] args) {

        Message msg = new Message("process it");
        Waiter waiter = new Waiter(msg);
        new Thread(waiter, "waiter").start();

        Waiter waiter1 = new Waiter(msg);
        new Thread(waiter1, "waiter1").start();

        Notifier notifier = new Notifier(msg);
        new Thread(notifier, "notifier").start();
        System.out.println("All the threads are started");
    }
}
```
在当前例子中，我们可以得到控制台中打出的结果：
```java
waiter1 waiting to get notified at time:1525606113222
waiter waiting to get notified at time:1525606113222
All the threads are started
notifier started
waiter1 waiter thread got notified at time:1525606114223
waiter1 processed: notifier Notifier work done
```

现在我们修改原来样例为：
```java
public class Notifier implements Runnable {
    private Message msg;
    public Notifier(Message msg) {this.msg = msg;}
    @Override
    public void run() {
        String name = Thread.currentThread().getName();
        System.out.println(name + " started");
        try {
            Thread.sleep(1000);
            synchronized (msg) {
                msg.setMsg(name + " Notifier work done");
//                msg.notify();
                msg.notifyAll();            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

得到预期的结果：
```java
waiter waiting to get notified at time:1525610336743
waiter1 waiting to get notified at time:1525610336743
All the threads are started
notifier started
waiter1 waiter thread got notified at time:1525610337744
waiter1 processed: notifier Notifier work done
waiter waiter thread got notified at time:1525610337744
waiter processed: notifier Notifier work done
```
##### 2.3.2 线程中断(interrupt)
在Java提供的线程支持类Thread中，有三个用于线程中断的方法：
```java
//中断线程。
public void interrupt();

//是一个静态方法，用于测试当前线程是否已经中断，并将线程的中断状态 清除。
//所以如果线程已经中断，调用两次interrupted，第二次时会返回false，因为第一次返回true后会清除中断状态。
public static boolean interrupted();

//测试线程是否已经中断。
public boolean isInterrupted();
```
以下是测试代码，你可以在[github][8]中看到：

打印线程如果没有被中断，会一直打印。
```java
public class InterruptThread implements Runnable {
    @Override
    public void run() {
        while (!Thread.currentThread().isInterrupted()) {     //如果当前线程未被中断，则执行打印工作
            System.out.println(Thread.currentThread().getName() + "打印中… …");

        }
        if (Thread.currentThread().isInterrupted()) {
            System.out.println("interrupted:" + Thread.interrupted());       //返回当前线程的状态，并清除状态
            System.out.println("isInterrupted:" + Thread.currentThread().isInterrupted());
        }
    }
}
```
测试类：
```java
public class InterruptedThreadLearn {
    public static void main(String[] args) {
        InterruptThread printer = new InterruptThread();
        Thread printerThread = new Thread(printer, "打印线程");
        printerThread.start();
        try {
            Thread.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("有紧急任务出现，需中断打印线程.");
        System.out.println("中断前的状态: printerThread.isInterrupted()=" + printerThread.isInterrupted());
        printerThread.interrupt();       // 中断打印线程
        System.out.println("中断后的状态：printerThread.isInterrupted()=" + printerThread.isInterrupted());
    }
}
```

result:
```java
打印线程打印中… …
打印线程打印中… …
打印线程打印中… …
有紧急任务出现，需中断打印线程.
打印线程打印中… …
中断前的状态: printerThread.isInterrupted()=false
打印线程打印中… …
中断后的状态：printerThread.isInterrupted()=true
interrupted:true
isInterrupted:false
```

##### 2.3.3 线程合并(join)
所谓合并，就是等待其它线程执行完，再执行当前线程，执行起来的效果就好像把其它线程合并到当前线程执行一样.
![img][6]
你可以在[github][7]上看到源码：
两个插件线程：
```java
public class PluginFirst implements Runnable {
    @Override
    public void run() {
        System.out.println("插件1开始安装.");
        System.out.println("安装中...");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("插件1完成安装.");
    }
}
```
```java
public class PluginSecond implements Runnable {

    @Override
    public void run() {
        System.out.println("插件2开始安装.");
        System.out.println("安装中...");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("插件2完成安装.");
    }

}
```
主线程(测试)
```java
public class ThreadJoinTest {
    public static void main(String[] args) {
        System.out.println("主线程开启...");
        Thread thread1 = new Thread(new PluginFirst());
        Thread thread2 = new Thread(new PluginSecond());
        try {
            thread1.start();   //开始插件1的安装
            thread1.join();       //等插件1的安装线程结束
            thread2.start();   //再开始插件2的安装
            thread2.join();       //等插件2的安装线程结束,才能回到主线程
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("主线程结束，程序安装完成！");
    }
}
```

##### 2.3.4 优先级(Priority)

线程优先级是指获得CPU资源的优先程序。优先级高的容易获得CPU资源，优先级底的较难获得CPU资源，表现出来的情况就是优先级越高执行的时间越多。Java中通过getPriority和setPriority方法获取和设置线程的优先级。Thread类提供了三个表示优先级的常量：MIN_PRIORITY优先级最低，为1；NORM_PRIORITY是正常的优先级；为5，MAX_PRIORITY优先级最高，为10。我们创建线程对象后，如果不显示的设置优先级的话，默认为5。
```java
thread1.setPriority(Thread.MAX_PRIORITY);
thread2.setPriority(8);
```

### 3. synchronized 的用法
synchronized是Java中的关键字，是一种同步锁。它修饰的对象有以下几种：

1. 修饰一个代码块，被修饰的代码块称为同步语句块，作用于传入的实例加锁，进入同步代码前要获得当前实例的锁
2. 修饰一个方法，被修饰的方法称为同步方法，作用于当前实例加锁，进入同步代码前要获得当前实例的锁
3. 修改一个静态的方法，作用于当前类对象加锁，进入同步代码前要获得当前类对象的锁
4. 修改一个类，指定加锁对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁。


#### 3.1 synchronized 修饰一个方法或代码块
我们可以把第一个第二个在同一个例子中展示(你可以在[github][9]中查看源码)。

Message是同步对象
```java
public class Message {

    private int number;


    public synchronized int methodA() {
        number = number +1;
        try {Thread.sleep(1000);
        } catch (InterruptedException e) {e.printStackTrace();}
        return number;
    }

    public synchronized int methodB() {
        number = number +1;
        try {Thread.sleep(1000);
        } catch (InterruptedException e) {e.printStackTrace();}
        return number;
    }

    public int methodC() {
        synchronized (this) {
            number = number +1;
            try {Thread.sleep(1000);
            } catch (InterruptedException e) {e.printStackTrace();}
            return number;
        }
    }

    public int methodD() {
        number = number +1;
        return number;
    }

}
```

下面我们创建若干个不同的thread用于获取不同的方法。
```java
public class SynchronizedThreadA implements Runnable {
    private Message object;
    public SynchronizedThreadA(Message object) {
        this.object = object;
    }

    @Override
    public void run() {
      System.out.println(Thread.currentThread().getName() + ", method a, number= "+ object.methodA());
    }
}

public class SynchronizedThreadB implements Runnable {
    private Message object;
    public SynchronizedThreadB(Message object) {
        this.object = object;
    }

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + ", method b, number= "+ object.methodB());
    }
}

public class SynchronizedThreadC implements Runnable {
...
}

public class SynchronizedThreadD implements Runnable {
...
}
```
注意，这里，我们主要是使用的是同一个对象object，在测试类中测试代码：
```java
public class SynchronizedInstanceTest {
    public static void main(String[] args) {
        Message object = new Message(); //不同的线程使用同一对象。
        for (int i = 0; i < 3; i++) {
            new Thread(new SynchronizedThreadA(object)).start();
        }
        for (int i = 0; i < 3; i++) {
            new Thread(new SynchronizedThreadB(object)).start();
        }
        for (int i = 0; i < 3; i++) {
            new Thread(new SynchronizedThreadC(object)).start();
        }
    }
}
```
会发现在控制台中，按照时间间隔打印出：
```java
Thread-0, method a, number= 1
Thread-8, method c, number= 2
Thread-7, method c, number= 3
Thread-6, method c, number= 4
Thread-5, method b, number= 5
Thread-4, method b, number= 6
Thread-3, method b, number= 7
Thread-2, method a, number= 8
Thread-1, method a, number= 9
```
你会发现，synchronized用代码块和在方法上的效果是一样的。
##### 3.1.1 synchronized 作用于不同对象
```java
public class SyncDiffMethodTest {
    public static void main(String[] args) {
        Message message1 = new Message();
        Message message2 = new Message();
        for (int i = 0;i<3 ;i++) {
            new Thread(new SynchronizedThreadA(message1)).start();
        }
        for (int i = 0;i<3 ;i++) {
            new Thread(new SynchronizedThreadB(message2)).start();
        }
    }
}
```
result:
```java
Thread-0, method a, number= 1
Thread-3, method b, number= 1
Thread-1, method a, number= 2
Thread-4, method b, number= 2
Thread-2, method a, number= 3
Thread-5, method b, number= 3
```
可以发现，作用与不同对象，并不会竞争同一把锁。

##### 3.1.2 synchronized 和 非synchronized方法

同样是获取同一个对象。
```java
public class SyncAndNoSyncMethodTest {
    public static void main(String[] args) {
        Message message = new Message();
        for (int i = 0; i < 3; i++) {
            new Thread(new SynchronizedThreadA(message)).start();
        }
        for (int i = 0; i < 10000; i++) {
            new Thread(new SynchronizedThreadD(message)).start();
        }
    }
}
```

result：
```java
···
Thread-2, method a, number= 8929
···
Thread-9937, method d, number= 9989
Thread-9928, method d, number= 9990
Thread-1, method a, number= 9990
Thread-0, method a, number= 9991
```
这时，d方法会被快速打印出来，也就是说，虽然，syncronized虽然会锁住对象，但是，并不影响非synchronized方法的执行。
Synchronized会使其他synchronized方法无法竞争到锁。

通过以上几个例子，我们知道多个线程调用同一个对象中的不同名称的synchronized同步方法或synchronized(this)同步代码块时，是同步的。
1. synchronized同步方法
- 对其它的synchronized同步方法或synchronized(this)同步代码块调用是堵塞状态；
- 同一时间只有一个线程执行synchronized同步方法中的代码。

2. synchronized(this)同步代码块
- 对其它的synchronized同步方法或synchronized(this)同步代码块调用是堵塞状态；
- 同一时间只有一个线程执行synchronized同步方法中的代码。

__
####修饰一个静态的方法
我们知道静态方法是属于类的而不属于对象的。同样的，synchronized修饰的静态方法锁定的是这个类的所有对象。
``` java
public class StaticMethodTest {
    public static void main(String[] args) {
        Message message1 = new Message();
        Message message2 = new Message();

        for (int i = 0; i < 3; i++) {
            new Thread(new SynchronizedThreadE(message1)).start();
        }
        for (int i = 0; i < 3; i++) {
            new Thread(new SynchronizedThreadE(message2)).start();
        }
    }
}
```
result:
```java
Thread-0, method e, number= 1
Thread-2, method e, number= 2
Thread-5, method e, number= 3
Thread-3, method e, number= 4
Thread-4, method e, number= 5
Thread-1, method e, number= 6
```
synchronized应用在static方法上，那是对当前对应的*.Class进行持锁。

#### synchronized(\*.class)代码块


在message中添加方法：
```Java
public static synchronized int methodE() {

    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    numbers = numbers +1;
    return numbers;
}

public int methodG() {
    synchronized (Message.class) {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        numbers = numbers +1;
        return numbers;
    }
}
```
在测试类中测试代码
```Java
public class SyncClassTest {
    public static void main(String[] args) {
        Message message1 = new Message();
        Message message2 = new Message();

        for (int i = 0; i < 3; i++) {
            new Thread(new SynchronizedThreadE(message1)).start();
        }
        for (int i = 0; i < 3; i++) {
            new Thread(new SynchronizedThreadG(message1)).start();
        }
        for (int i = 0; i < 3; i++) {
            new Thread(new SynchronizedThreadG(message2)).start();
        }
    }
}
```
result:
```Java
Thread-1, method e, number= 1
Thread-8, method g, number= 2
Thread-7, method g, number= 3
Thread-5, method g, number= 4
Thread-6, method g, number= 5
Thread-4, method g, number= 6
Thread-3, method g, number= 7
Thread-0, method e, number= 8
Thread-2, method e, number= 9
```
同步synchronized(\*.class)代码块的作用其实和synchronized static方法作用一样。Class锁对类的所有对象实例起作用。

但是，请注意：

<font color=#117A65>
Java语法规定，任何线程执行同步方法、同步代码块之前，必须先获取对应的监视器。并且监听器this 和 \*.class 是不同的。
this 是对用方法的对象本身 class 是该类本身(只有监听器相同锁才会起作用)。
当synchronized作用于静态方法时，其锁就是当前类的class对象锁。由于静态成员不专属于任何一个实例对象，是类成员，因此通过class对象锁可以控制静态 成员的并发操作。需要注意的是如果一个线程A调用一个实例对象的非static synchronized方法，而线程B需要调用这个实例对象所属类的静态 synchronized方法，是允许的，不会发生互斥现象，因为访问静态 synchronized 方法占用的锁是当前类的class对象，而访问非静态 synchronized 方法占用的锁是当前实例对象锁。
</font>





### 4. synchronized 底层实现

感谢[深入理解Java并发之synchronized实现原理][11]

Java 虚拟机中的同步(Synchronization)基于进入和退出管程(Monitor)对象实现， 无论是显式同步(有明确的 monitorenter 和 monitorexit 指令,即同步代码块)还是隐式同步都是如此。在 Java 语言中，同步用的最多的地方可能是被 synchronized 修饰的同步方法。同步方法 并不是由 monitorenter 和 monitorexit 指令来实现同步的，而是由方法调用指令读取运行时常量池中方法的 ACC_SYNCHRONIZED 标志来隐式实现的。

#### 4.1 理解Java对象头与Monitor
在JVM中，对象在内存中的布局分为三块区域：对象头、实例数据和对齐填充。

- 实例变量：存放类的属性数据信息，包括父类的属性信息，如果是数组的实例部分还包括数组的长度，这部分内存按4字节对齐。
- 填充数据：由于虚拟机要求对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐，这点了解即可。


|虚拟机位数	|头对象结构	            |说明                                                              |
|-----------|-----------------------|------------------------------------------------------------------|
|32/64bit   |	Mark Word	            |存储对象的hashCode、锁信息或分代年龄或GC标志等信息。                  |
|32/64bit	  |Class Metadata Address	|类型指针指向对象的类元数据，JVM通过这个指针确定该对象是哪个类的实例。   |


其中Mark Word在默认情况下存储着对象的HashCode、分代年龄、锁标记位等以下是32位JVM的Mark Word默认存储结构

|锁状态	 |25bit	         |4bit	         |1bit是否是偏向锁	|2bit 锁标志位 |
|--------|----------------|--------------|-----------------|--------------|
|无锁状态	|对象HashCode	   |对象分代年龄	 |0	               |01            |

由于对象头的信息是与对象自身定义的数据没有关系的额外存储成本，因此考虑到JVM的空间效率，Mark Word 被设计成为一个非固定的数据结构，以便存储更多有效的数据，它会根据对象本身的状态复用自己的存储空间，如32位JVM下，除了上述列出的Mark Word默认存储结构外，还有如下可能变化的结构：

![img][12]

其中轻量级锁和偏向锁是Java 6 对 synchronized 锁进行优化后新增加的，稍后我们会简要分析。这里我们主要分析一下重量级锁也就是通常说synchronized的对象锁，锁标识位为10，其中指针指向的是monitor对象（也称为管程或监视器锁）的起始地址。每个对象都存在着一个 monitor 与之关联，对象与其 monitor 之间的关系有存在多种实现方式，如monitor可以与对象一起创建销毁或当线程试图获取对象锁时自动生成，但当一个 monitor 被某个线程持有后，它便处于锁定状态。在Java虚拟机(HotSpot)中，monitor是由ObjectMonitor实现的，其主要数据结构如下（位于HotSpot虚拟机源码ObjectMonitor.hpp文件，C++实现的）
<!-- ```c
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //记录个数
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
``` -->

ObjectMonitor中有两个队列，\_WaitSet 和 \_EntryList，用来保存ObjectWaiter对象列表( 每个等待锁的线程都会被封装成ObjectWaiter对象)，\_owner指向持有ObjectMonitor对象的线程，当多个线程同时访问一段同步代码时，首先会进入 \_EntryList 集合，当线程获取到对象的monitor 后进入 \_Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1，若线程调用 wait() 方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSe t集合中等待被唤醒。若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)。

### 4.2 synchronized代码块底层原理
```Java
public class SyncCodeBlock {
   public int i;
   public void syncTask(){
       //同步代码库
       synchronized (this){
           i++;
       }
   }
}
```
编译上述代码并使用javap反编译后得到字节码如下:
```Java
//===========主要看看syncTask方法实现================
  public void syncTask();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter  //注意此处，进入同步方法
         4: aload_0
         5: dup
         6: getfield      #2             // Field i:I
         9: iconst_1
        10: iadd
        11: putfield      #2            // Field i:I
        14: aload_1
        15: monitorexit   //注意此处，退出同步方法
        16: goto          24
        19: astore_2
        20: aload_1
        21: monitorexit //注意此处，退出同步方法
        22: aload_2
        23: athrow
        24: return
      Exception table:
      //省略其他字节码.......
```
核心：
```java
3: monitorenter  //进入同步方法
//..........省略其他  
15: monitorexit   //退出同步方法
16: goto          24
//省略其他.......
21: monitorexit //退出同步方法
```

从字节码中可知同步语句块的实现使用的是`monitorenter`和`monitorexit`指令，其中`monitorenter`指令指向同步代码块的开始位置，`monitorexit`指令则指明同步代码块的结束位置，当执行`monitorenter`指令时，当前线程将试图获取 `objectref`(即对象锁) 所对应的`monitor`的持有权，当`objectref`` monitor`的进入计数器为 0，那线程可以成功取得`monitor`，并将计数器值设置为 1，取锁成功。如果当前线程已经拥有 `objectref` 的 `monitor` 的持有权，那它可以重入这个 `monitor` (关于重入性稍后会分析)，重入时计数器的值也会加 1。倘若其他线程已经拥有 `objectref` 的 `monitor` 的所有权，那当前线程将被阻塞，直到正在执行线程执行完毕，即`monitorexit`指令被执行，执行线程将释放 `monitor`(锁)并设置计数器值为0 ，其他线程将有机会持有 `monitor` 。值得注意的是编译器将会确保无论方法通过何种方式完成，方法中调用过的每条 `monitorenter` 指令都有执行其对应 `monitorexit` 指令，而无论这个方法是正常结束还是异常结束。为了保证在方法异常完成时 `monitorenter` 和 `monitorexit` 指令依然可以正确配对执行，编译器会自动产生一个异常处理器，这个异常处理器声明可处理所有的异常，它的目的就是用来执行 `monitorexit` 指令。从字节码中也可以看出多了一个`monitorexit`指令，它就是异常结束时被执行的释放`monitor` 的指令。

#### 4.3 synchronized方法底层原理

方法级的同步是隐式，即无需通过字节码指令来控制的，它实现在方法调用和返回操作之中。JVM可以从方法常量池中的方法表结构`(method_info Structure)`中的 `ACC_SYNCHRONIZED`访问标志区分一个方法是否同步方法。当方法调用时，调用指令将会 检查方法的 `ACC_SYNCHRONIZED` 访问标志是否被设置，如果设置了，执行线程将先持有`monitor`（虚拟机规范中用的是管程一词）， 然后再执行方法，最后再方法完成(无论是正常完成还是非正常完成)时释放`monitor`。在方法执行期间，执行线程持有了`monitor`，其他任何线程都无法再获得同一个`monitor`。如果一个同步方法执行期间抛 出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的`monitor`将在异常抛到同步方法之外时自动释放。下面我们看看字节码层面如何实现。
```java
public class SyncMethod {
   public int i;
   public synchronized void syncTask(){
           i++;
   }
}
```
同样，编译之后，主要字节码如下：
```java
//==================syncTask方法======================
  public synchronized void syncTask();
    descriptor: ()V
    //方法标识ACC_PUBLIC代表public修饰，ACC_SYNCHRONIZED指明该方法为同步方法
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #2                  // Field i:I
         5: iconst_1
         6: iadd
         7: putfield      #2                  // Field i:I
        10: return
      LineNumberTable:
        line 12: 0
        line 13: 10
```

从字节码中可以看出，`synchronized`修饰的方法并没有`monitorenter`指令和`monitorexit`指令，取得代之的确实是`ACC_SYNCHRONIZED`标识，该标识指明了该方法是一个同步方法，JVM通过该`ACC_SYNCHRONIZED`访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。这便是`synchronized`锁在同步代码块和同步方法上实现的基本原理。同时我们还必须注意到的是在Java早期版本中，`synchronized`属于重量级锁，效率低下，因为监视器锁`（monitor）`是依赖于底层的操作系统的`Mutex Lock`来实现的，而操作系统实现线程之间的切换时需要从用户态转换到核心态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，这也是为什么早期的`synchronized`效率低的原因。庆幸的是在`Java 6`之后Java官方对从JVM层面对`synchronized`较大优化，所以现在的`synchronized`锁效率也优化得很不错了，`Java 6`之后，为了减少获得锁和释放锁所带来的性能消耗，引入了轻量级锁和偏向锁，接下来我们将简单了解一下Java官方在JVM层面对`synchronized`锁的优化。

### 5 Java虚拟机对synchronized的优化
锁的状态总共有四种，无锁状态、偏向锁、轻量级锁和重量级锁。随着锁的竞争，锁可以从偏向锁升级到轻量级锁，再升级的重量级锁，但是锁的升级是单向的，也就是说只能从低到高升级，不会出现锁的降级。

#### 5.1 偏向锁
偏向锁是Java 6之后加入的新锁，它是一种针对加锁操作的优化手段，经过研究发现，在大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，因此为了减少同一线程获取锁(会涉及到一些CAS操作,耗时)的代价而引入偏向锁。偏向锁的核心思想是，如果一个线程获得了锁，那么锁就进入偏向模式，此时Mark Word 的结构也变为偏向锁结构，当这个线程再次请求锁时，无需再做任何同步操作，即获取锁的过程，这样就省去了大量有关锁申请的操作，从而也就提供程序的性能。所以，对于没有锁竞争的场合，偏向锁有很好的优化效果，毕竟极有可能连续多次是同一个线程申请相同的锁。但是对于锁竞争比较激烈的场合，偏向锁就失效了，因为这样场合极有可能每次申请锁的线程都是不相同的，因此这种场合下不应该使用偏向锁，否则会得不偿失，需要注意的是，偏向锁失败后，并不会立即膨胀为重量级锁，而是先升级为轻量级锁。下面我们接着了解轻量级锁。

#### 5.2 轻量级锁
倘若偏向锁失败，虚拟机并不会立即升级为重量级锁，它还会尝试使用一种称为轻量级锁的优化手段(1.6之后加入的)，此时Mark Word 的结构也变为轻量级锁的结构。轻量级锁能够提升程序性能的依据是“对绝大部分的锁，在整个同步周期内都不存在竞争”，注意这是经验数据。需要了解的是，轻量级锁所适应的场景是线程交替执行同步块的场合，如果存在同一时间访问同一锁的场合，就会导致轻量级锁膨胀为重量级锁。

#### 5.3 自旋锁
轻量级锁失败后，虚拟机为了避免线程真实地在操作系统层面挂起，还会进行一项称为自旋锁的优化手段。这是基于在大多数情况下，线程持有锁的时间都不会太长，如果直接挂起操作系统层面的线程可能会得不偿失，毕竟操作系统实现线程之间的切换时需要从用户态转换到核心态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，因此自旋锁会假设在不久将来，当前的线程可以获得锁，因此虚拟机会让当前想要获取锁的线程做几个空循环(这也是称为自旋的原因)，一般不会太久，可能是50个循环或100循环，在经过若干次循环后，如果得到锁，就顺利进入临界区。如果还不能获得锁，那就会将线程在操作系统层面挂起，这就是自旋锁的优化方式，这种方式确实也是可以提升效率的。最后没办法也就只能升级为重量级锁了。

#### 5.4 锁消除
消除锁是虚拟机另外一种锁的优化，这种优化更彻底，Java虚拟机在JIT编译时(可以简单理解为当某段代码即将第一次被执行时进行编译，又称即时编译)，通过对运行上下文的扫描，去除不可能存在共享资源竞争的锁，通过这种方式消除没有必要的锁，可以节省毫无意义的请求锁时间，如下StringBuffer的append是一个同步方法，但是在add方法中的StringBuffer属于一个局部变量，并且不会被其他线程所使用，因此StringBuffer不可能存在共享资源竞争的情景，JVM会自动将其锁消除。
```Java
public class StringBufferRemoveSync {

    public void add(String str1, String str2) {
        //StringBuffer是线程安全,由于sb只会在append方法中使用,不可能被其他线程引用
        //因此sb属于不可能共享的资源,JVM会自动消除内部的锁
        StringBuffer sb = new StringBuffer();
        sb.append(str1).append(str2);
    }
    public static void main(String[] args) {
        StringBufferRemoveSync rmsync = new StringBufferRemoveSync();
        for (int i = 0; i < 10000000; i++) {
            rmsync.add("abc", "123");
        }
    }
}
```
### 6 synchronized的几个特点
#### 6.1 synchronized的可重入性
从互斥锁的设计上来说，当一个线程试图操作一个由其他线程持有的对象锁的临界资源时，将会处于阻塞状态，但当一个线程再次请求自己持有对象锁的临界资源时，这种情况属于重入锁，请求将会成功，在java中synchronized是基于原子性的内部锁机制，是可重入的，因此在一个线程调用synchronized方法的同时在其方法体内部调用该对象另一个synchronized方法，也就是说一个线程得到一个对象锁后再次请求该对象锁，是允许的，这就是synchronized的可重入性。
```java
public class AccountingSync implements Runnable{
    static AccountingSync instance=new AccountingSync();
    static int i=0;
    static int j=0;
    @Override
    public void run() {
        for(int j=0;j<1000000;j++){

            //this,当前实例对象锁
            synchronized(this){
                i++;
                increase();//synchronized的可重入性
            }
        }
    }

    public synchronized void increase(){
        j++;
    }


    public static void main(String[] args) throws InterruptedException {
        Thread t1=new Thread(instance);
        Thread t2=new Thread(instance);
        t1.start();t2.start();
        t1.join();t2.join();
        System.out.println(i);
    }
}
```

#### 6.2 等待唤醒机制与synchronized
所谓等待唤醒机制本篇主要指的是`notify/notifyAll`和`wait`方法，在使用这3个方法时，必须处于`synchronized`代码块或者`synchronized`方法中，否则就会抛出`IllegalMonitorStateException`异常，这是因为调用这几个方法前必须拿到当前对象的监视器`monitor`对象，也就是说`notify/notifyAll`和`wait`方法依赖于`monitor`对象，在前面的分析中，我们知道`monitor` 存在于对象头的`Mark Word` 中(存储`monitor`引用指针)，而`synchronized`关键字可以获取 `monitor` ，这也就是为什么`notify/notifyAll`和`wait`方法必须在`synchronized`代码块或者`synchronized`方法调用的原因。

需要特别理解的一点是，与`sleep`方法不同的是`wait`方法调用完成后，线程将被暂停，但`wait`方法将会释放当前持有的监视器锁(monitor)，直到有线程调用`notify/notifyAll`方法后方能继续执行，而`sleep`方法只让线程休眠并不释放锁。同时`notify/notifyAll`方法调用后，并不会马上释放监视器锁，而是在相应的`synchronized(){}/synchronized`方法执行结束后才自动释放锁。

#### 6.3 线程中断与synchronized
```java
public class InterruptThreadTest {
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread() {
            @Override
            public void run() {
                //while在try中，通过异常中断就可以退出run循环
                try {
                    while (true) {
                        //当前线程处于阻塞状态，异常必须捕捉处理，无法往外抛出
                        TimeUnit.SECONDS.sleep(2);
                    }
                } catch (InterruptedException e) {
                    System.out.println("Interruted When Sleep");
                    boolean interrupt = this.isInterrupted();
                    //中断状态被复位
                    System.out.println("interrupt:" + interrupt);
                }
            }
        };
        t1.start();
        TimeUnit.SECONDS.sleep(2);
        //中断处于阻塞状态的线程
        t1.interrupt();

    }
}
```
output:
```Java
Interruted When Sleep
interrupt:false
```
如上述代码所示，我们创建一个线程，并在线程中调用了sleep方法从而使用线程进入阻塞状态，启动线程后，**调用线程实例对象的interrupt方法中断阻塞异常，并抛出InterruptedException异常，此时中断状态也将被复位**。

除了阻塞中断的情景，我们还可能会遇到处于运行期且非阻塞的状态的线程，这种情况下，直接调用Thread.interrupt()中断线程是不会得到任响应的。
```Java
public class InterruptThread2 {
    public static void main(String[] args) throws InterruptedException {
        Thread t1=new Thread(){
            @Override
            public void run(){
                while(true){System.out.println("未被中断");}
            }
        };
        t1.start();
//        TimeUnit.SECONDS.sleep(2);
//        Thread.sleep(100000);
        t1.interrupt();
    }
}
```
output:
```
  ···
  未被中断
  ···
```
虽然我们调用了interrupt方法，但线程t1并未被中断，因为处于非阻塞状态的线程需要我们手动进行中断检测并结束程序
```Java
public class RunningInterruptThread {
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread() {
            @Override
            public void run() {
                while (true) {
                    //判断当前线程是否被中断
                    if (this.isInterrupted()) {
                        System.out.println("线程中断");
                        break;
                    }
                }
                System.out.println("已跳出循环,线程中断!");
            }
        };
        t1.start();
        TimeUnit.SECONDS.sleep(2);
        t1.interrupt();
    }
}
```
是的，我们在代码中使用了实例方法`isInterrupted`判断线程是否已被中断，如果被中断将跳出循环以此结束线程,注意非阻塞状态调用`interrupt()`并不会导致中断状态重置。综合所述，可以简单总结一下中断两种情况，一种是当线程处于阻塞状态或者试图执行一个阻塞操作时，我们可以使用实例方法`interrupt()`进行线程中断，执行中断操作后将会抛出`interruptException`异常(该异常必须捕捉无法向外抛出)并将中断状态复位，另外一种是当线程处于运行状态时，我们也可调用实例方法interrupt()进行线程中断，但同时必须手动判断中断状态，并编写中断线程的代码(其实就是结束run方法体的代码).

有时候需要兼顾这两种情况：
```Java
public void run(){
    try {
    //判断当前线程是否已中断,注意interrupted方法是静态的,执行后会对中断状态进行复位
    while (!Thread.interrupted()) {
        TimeUnit.SECONDS.sleep(2);
    }
    } catch (InterruptedException e) {

    }
}
```
#### 6.4 synchronized 与锁等待
事实上线程的中断操作对于正在等待获取的锁对象的synchronized方法或者代码块并不起作用，也就是对于synchronized来说，如果一个线程在等待锁，那么结果只有两种，要么它获得这把锁继续执行，要么它就保存等待，即使调用中断线程的方法，也不会生效。
```Java
public class SynchronizedBlocked implements Runnable {

    public synchronized void f() {
        System.out.println("Trying to call f()");
        while (true) // Never releases lock
            Thread.yield();
    }
    /**
     * 在构造器中创建新线程并启动获取对象锁
     */
    public SynchronizedBlocked() {
        //该线程已持有当前实例锁
        new Thread() {
            public void run() {
                f(); // Lock acquired by this thread
            }
        }.start();
    }
    public void run() {
        //中断判断
        while (true) {
            if (Thread.interrupted()) {
                System.out.println("中断线程!!");
                break;
            } else {
                f();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        SynchronizedBlocked sync = new SynchronizedBlocked();
        Thread t = new Thread(sync);
        //启动后调用f()方法,无法获取当前实例锁处于等待状态
        t.start();
        TimeUnit.SECONDS.sleep(1);
        //中断线程,无法生效
        t.interrupt();
    }
}
```
在`SynchronizedBlocked`构造函数中创建一个新线程并启动获取调用`f()`获取到当前实例锁，由于`SynchronizedBlocked`自身也是线程，启动后在其`run`方法中也调用了`f()`，但由于对象锁被其他线程占用，导致t线程只能等到锁，此时我们调用了`t.interrupt()`;但并不能中断线程。













---

引用文章：

[编程思想之多线程与多进程(1)——以操作系统的角度述说线程与进程][1]

[编程思想之多线程与多进程(3)——Java中的多线程][3]

[Java多线程中wait, notify and notifyAll的使用][4]

[Java中Synchronized的用法][10]

[深入理解Java并发之synchronized实现原理][11]

[1]:https://blog.csdn.net/luoweifu/article/details/46595285
[2]:https://img-blog.csdn.net/20150623201557825?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHVvd2VpZnU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center
[3]:https://blog.csdn.net/luoweifu/article/details/46673975
[4]:https://blog.csdn.net/luoweifu/article/details/46664809
[5]:https://github.com/twentyworld/learn/tree/master/JDKlearn/src/main/java/com/thread/waitLearn
[6]:https://img-blog.csdn.net/20150628204041010
[7]:https://github.com/twentyworld/learn/tree/master/JDKlearn/src/main/java/com/thread/joinLearn
[8]:https://github.com/twentyworld/learn/tree/master/JDKlearn/src/main/java/com/thread/interruptedLearn
[9]:https://github.com/twentyworld/learn/tree/master/JDKlearn/src/main/java/com/synchronizedDemo
[10]:https://blog.csdn.net/luoweifu/article/details/46613015
[11]:https://blog.csdn.net/javazejian/article/details/72828483
[12]:https://img-blog.csdn.net/20170603172215966?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamF2YXplamlhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast


[]
