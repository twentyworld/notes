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


---
你可以在[github][5]上看到源码：

一个java bean类，线程将会使用它并调用wait和notify方法。
```
public class Message {
    private String msg;
    public Message(String str){this.msg=str;}
    public String getMsg() {return msg;}
    public void setMsg(String str) {this.msg=str;}
}
```
一个Waiter类，等待其它的线程调用notify方法以唤醒线程完成处理。注意等待线程必须通过加synchronized同步锁拥有Message对象的监视器。
```
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
```
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
```
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
```
waiter1 waiting to get notified at time:1525606113222
waiter waiting to get notified at time:1525606113222
All the threads are started
notifier started
waiter1 waiter thread got notified at time:1525606114223
waiter1 processed: notifier Notifier work done
```

现在我们修改原来样例为：
```
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
```
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
```
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
```
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
```
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
```
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
```
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
```
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
```
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
```
thread1.setPriority(Thread.MAX_PRIORITY);
thread2.setPriority(8);
```

### 3. synchronized 的用法
synchronized是Java中的关键字，是一种同步锁。它修饰的对象有以下几种：

1. 修饰一个代码块，被修饰的代码块称为同步语句块，其作用的范围是大括号{}括起来的代码，作用的对象是调用这个代码块的对象；
2. 修饰一个方法，被修饰的方法称为同步方法，其作用的范围是整个方法，作用的对象是调用这个方法的对象；
3. 修改一个静态的方法，其作用的范围是整个静态方法，作用的对象是这个类的所有对象；
4. 修改一个类，其作用的范围是synchronized后面括号括起来的部分，作用主的对象是这个类的所有对象。


#### 3.1 synchronized 修饰一个方法或代码块
我们可以把第一个第二个在同一个例子中展示(你可以在[github][9]中查看源码)。

Message是同步对象
```
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
```
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
```
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
```
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
```
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
```
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
```
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
```
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

####修饰一个静态的方法
我们知道静态方法是属于类的而不属于对象的。同样的，synchronized修饰的静态方法锁定的是这个类的所有对象。
```
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
```
Thread-0, method e, number= 1
Thread-2, method e, number= 2
Thread-5, method e, number= 3
Thread-3, method e, number= 4
Thread-4, method e, number= 5
Thread-1, method e, number= 6
```
虽然不是同一个对象，但是，













































引用：

[编程思想之多线程与多进程(1)——以操作系统的角度述说线程与进程][1]

[编程思想之多线程与多进程(3)——Java中的多线程][3]

[Java多线程中wait, notify and notifyAll的使用][4]

[Java中Synchronized的用法][10]

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

[]
