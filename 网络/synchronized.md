# synchronized
---
## 1. 进程与线程
> 本节感谢[编程思想之多线程与多进程(1)——以操作系统的角度述说线程与进程][1]

### 1.1 任务调度
大部分操作系统(如Windows、Linux)的任务调度是采用时间片轮转的抢占式调度方式，也就是说一个任务执行一小段时间后强制暂停去执行下一个任务，每个任务轮流执行。

任务执行的一小段时间叫做时间片，任务正在执行时的状态叫运行状态，任务执行一段时间后强制暂停去执行下一个任务，被暂停的任务就处于就绪状态等待下一个属于它的时间片的到来。这样每个任务都能得到执行，由于CPU的执行效率非常高，时间片非常短，在各个任务之间快速地切换，给人的感觉就是多个任务在“同时进行”，这也就是我们所说的并发(别觉得并发有多高深，它的实现很复杂，但它的概念很简单，就是一句话：多个任务同时执行)。

### 1.2 进程
我们都知道计算机的核心是CPU，它承担了所有的计算任务；而操作系统是计算机的管理者，它负责任务的调度、资源的分配和管理，统领整个计算机硬件；应用程序侧是具有某种功能的程序，程序是运行于操作系统之上的。

进程是一个具有一定独立功能的程序在一个数据集上的一次动态执行的过程，是操作系统进行资源分配和调度的一个独立单位，是应用程序运行的载体。进程是一种抽象的概念，从来没有统一的标准定义。进程一般由程序、数据集合和进程控制块三部分组成。程序用于描述进程要完成的功能，是控制进程执行的指令集；数据集合是程序在执行时所需要的数据和工作区；程序控制块(Program Control Block，简称PCB)，包含进程的描述信息和控制信息，是进程存在的唯一标志。

进程具有的特征：
1. 动态性：进程是程序的一次执行过程，是临时的，有生命期的，是动态产生，动态消亡的；
2. 并发性：任何进程都可以同其他进程一起并发执行；
3. 独立性：进程是系统进行资源分配和调度的一个独立单位；
4. 结构性：进程由程序、数据和进程控制块三部分组成。

### 1.3 线程
在早期的操作系统中并没有线程的概念，进程是能拥有资源和独立运行的最小单位，也是程序执行的最小单位。任务调度采用的是时间片轮转的抢占式调度方式，而进程是任务调度的最小单位，每个进程有各自独立的一块内存，使得各个进程之间内存地址相互隔离。

后来，随着计算机的发展，对CPU的要求越来越高，进程之间的切换开销较大，已经无法满足越来越复杂的程序的要求了。于是就发明了线程，线程是程序执行中一个单一的顺序控制流程，是程序执行流的最小单元，是处理器调度和分派的基本单位。一个进程可以有一个或多个线程，各个线程之间共享程序的内存空间(也就是所在进程的内存空间)。一个标准的线程由线程ID、当前指令指针(PC)、寄存器和堆栈组成。而进程由内存空间(代码、数据、进程空间、打开的文件)和一个或多个线程组成。

### 1.4 多核与多线程
多核(心)处理器是指在一个处理器上集成多个运算核心从而提高计算能力，也就是有多个真正并行计算的处理核心，每一个处理核心对应一个内核线程。内核线程（Kernel Thread， KLT）就是直接由操作系统内核支持的线程，这种线程由内核来完成线程切换，内核通过操作调度器对线程进行调度，并负责将线程的任务映射到各个处理器上。一般一个处理核心对应一个内核线程，比如单核处理器对应一个内核线程，双核处理器对应两个内核线程，四核处理器对应四个内核线程。

现在的电脑一般是双核四线程、四核八线程，是采用超线程技术将一个物理处理核心模拟成两个逻辑处理核心，对应两个内核线程，所以在操作系统中看到的CPU数量是实际物理CPU数量的两倍。

超线程技术就是利用特殊的硬件指令，把一个物理芯片模拟成两个逻辑处理核心，让单个处理器都能使用线程级并行计算，进而兼容多线程操作系统和软件，减少了CPU的闲置时间，提高的CPU的运行效率。这种超线程技术(如双核四线程)由处理器硬件的决定，同时也需要操作系统的支持才能在计算机中表现出来。

程序一般不会直接去使用内核线程，而是去使用内核线程的一种高级接口——轻量级进程（Light Weight Process，LWP），轻量级进程就是我们通常意义上所讲的线程(我们在这称它为用户线程)，由于每个轻量级进程都由一个内核线程支持，因此只有先支持内核线程，才能有轻量级进程。用户线程与内核线程的对应关系有三种模型：一对一模型、多对一模型、多对多模型，在这以4个内核线程、3个用户线程为例对三种模型进行说明。

#### 1.4.1 一对一模型
对于一对一模型来说，一个用户线程就唯一地对应一个内核线程(反过来不一定成立，一个内核线程不一定有对应的用户线程)。这样，如果CPU没有采用超线程技术(如四核四线程的计算机)，一个用户线程就唯一地映射到一个物理CPU的线程，线程之间的并发是真正的并发。一对一模型使用户线程具有与内核线程一样的优点，一个线程因某种原因阻塞时其他线程的执行不受影响；此处，一对一模型也可以让多线程程序在多处理器的系统上有更好的表现。

但一对一模型也有两个缺点：1.许多操作系统限制了内核线程的数量，因此一对一模型会使用户线程的数量受到限制；2.许多操作系统内核线程调度时，上下文切换的开销较大，导致用户线程的执行效率下降。

#### 1.4.2 多对一模型
多对一模型将多个用户线程映射到一个内核线程上，线程之间的切换由用户态的代码来进行，因此相对一对一模型，多对一模型的线程切换速度要快许多；此外，多对一模型对用户线程的数量几乎无限制。但多对一模型也有两个缺点：1.如果其中一个用户线程阻塞，那么其它所有线程都将无法执行，因为此时内核线程也随之阻塞了；2.在多处理器系统上，处理器数量的增加对多对一模型的线程性能不会有明显的增加，因为所有的用户线程都映射到一个处理器上了。

#### 1.4.3 多对多模型

多对多模型结合了一对一模型和多对一模型的优点，将多个用户线程映射到多个内核线程上。多对多模型的优点有：1.一个用户线程的阻塞不会导致所有线程的阻塞，因为此时还有别的内核线程被调度来执行；2.多对多模型对用户线程的数量没有限制；3.在多处理器的操作系统中，多对多模型的线程也能得到一定的性能提升，但提升的幅度不如一对一模型的高。

在现在流行的操作系统中，大都采用多对多的模型。

![多对多模型][2]
### 1.5 线程的生命周期
当线程的数量小于处理器的数量时，线程的并发是真正的并发，不同的线程运行在不同的处理器上。但当线程的数量大于处理器的数量时，线程的并发会受到一些阻碍，此时并不是真正的并发，因为此时至少有一个处理器会运行多个线程。

在单个处理器运行多个线程时，并发是一种模拟出来的状态。操作系统采用时间片轮转的方式轮流执行每一个线程。现在，几乎所有的现代操作系统采用的都是时间片轮转的抢占式调度方式，如我们熟悉的Unix、Linux、Windows及Mac OS X等流行的操作系统。

我们知道线程是程序执行的最小单位，也是任务执行的最小单位。在早期只有进程的操作系统中，进程有五种状态，创建、就绪、运行、阻塞(等待)、退出。早期的进程相当于现在的只有单个线程的进程，那么现在的多线程也有五种状态，现在的多线程的生命周期与早期进程的生命周期类似。

### 1.6 线程优先级与线程安全
#### 1.6.1 线程优先级
现在主流操作系统(如Windows、Linux、Mac OS X)的任务调度除了具有前面提到的时间片轮转的特点外，还有优先级调度(Priority Schedule)的特点。优先级调度决定了线程按照什么顺序轮流执行，在具有优先级调度的系统中，线程拥有各自的线程优先级(Thread Priority)。具有高优先级的线程会更早地执行，而低优先级的线程通常要等没有更高优先级的可执行线程时才会被执行。

线程的优先级可以由用户手动设置，此外系统也会根据不同情形调整优先级。通常情况下，频繁地进入等待状态(进入等待状态会放弃之前仍可占用的时间份额)的线程(如IO线程)，比频繁进行大量计算以至于每次都把所有时间片全部用尽的线程更受操作系统的欢迎。因为频繁进入等待的线程只会占用很少的时间，这样操作系统可以处理更多的任务。我们把频繁等待的线程称之为IO密集型线程(IO Bound Thread)，而把很少等待的线程称之为CPU密集型线程(CPU Bound Thread)。IO密集型线程总是比CPU密集型线程更容易得到优先级的提升。
#### 1.6.2 线程饿死:
在优先级调度下，容易出现一种线程饿死的现象。一个线程饿死是说它的优先级较低，在它执行之前总是有比它优先级更高的线程等待执行，因此这个低优先级的线程始终得不到执行。当CPU密集型的线程优先级较高时，其它低优先级的线程就很可能出现饿死的情况；当IO密集型线程优先级较高时，其它线程相对不容易造成饿死的善，因为IO线程有大量的等待时间。为了避免线程饿死，调度系统通常会逐步提升那些等待了很久而得不到执行的线程的优先级。这样，一个线程只要它等待了足够长的时间，其优先级总会被提升到可以让它执行的程度，也就是说这种情况下线程始终会得到执行，只是时间的问题。

在优先级调度环境下，线程优先级的改变有三种方式：
1. 用户指定优先级；
2. 根据进入等待状态的频繁程度提升或降低优先级(由操作系统完成)；
3. 长时间得不到执行而被提升优先级。

#### 1.6.3 线程安全与锁

所谓同步(synchronization)就是指一个线程访问数据时，其它线程不得对同一个数据进行访问，即同一时刻只能有一个线程访问该数据，当这一线程访问结束时其它线程才能对这它进行访问。同步最常见的方式就是使用锁(Lock)，也称为线程锁。锁是一种非强制机制，每一个线程在访问数据或资源之前，首先试图获取(Acquire)锁，并在访问结束之后释放(Release)锁。在锁被占用时试图获取锁，线程会进入等待状态，直到锁被释放再次变为可用。

##### 二元信号量
二元信号量(Binary Semaphore)是一种最简单的锁，它有两种状态：占用和非占用。它适合只能被唯一一个线程独占访问的资源。当二元信号量处于非占用状态时，第一个试图获取该二元信号量锁的线程会获得该锁，并将二元信号量锁置为占用状态，之后其它试图获取该二元信号量的线程会进入等待状态，直到该锁被释放。

##### 信号量
多元信号量允许多个线程访问同一个资源，多元信号量简称信号量(Semaphore)，对于允许多个线程并发访问的资源，这是一个很好的选择。一个初始值为N的信号量允许N个线程并发访问。线程访问资源时首先获取信号量锁，进行如下操作：
1. 将信号量的值减1；
2. 如果信号量的值小于0，则进入等待状态，否则继续执行；
访问资源结束之后，线程释放信号量锁，进行如下操作：
1. 将信号量的值加1；
2. 如果信号量的值小于1(等于0)，唤醒一个等待中的线程；

##### 互斥量
互斥量(Mutex)和二元信号量类似，资源仅允许一个线程访问。与二元信号量不同的是，信号量在整个系统中可以被任意线程获取和释放，也就是说，同一个信号量可以由一个线程获取而由另一线程释放。而互斥量则要求哪个线程获取了该互斥量锁就由哪个线程释放，其它线程越俎代庖释放互斥量是无效的。

##### 临界区
临界区(Critical Section)是一种比互斥量更加严格的同步手段。互斥量和信号量在系统的任何进程都是可见的，也就是说一个进程创建了一个互斥量或信号量，另一进程试图获取该锁是合法的。而临界区的作用范围仅限于本进程，其它的进程无法获取该锁。除此之处，临界区与互斥量的性质相同。
##### 读写锁
读写锁(Read-Write Lock)允许多个线程同时对同一个数据进行读操作，而只允许一个线程进行写操作。这是因为读操作不会改变数据的内容，是安全的；而写操作会改变数据的内容，是不安全的。对同一个读写锁，有两种获取方式：共享的(Shared)和独占的(Exclusive)。当锁处于自由状态时，试图以任何一种方式获取锁都能成功，并将锁置为对应的状态；如果锁处于共享状态，其它线程以共享方式获取该锁，仍然能成功，此时该锁分配给了多个线程；如果其它线程试图如独占的方式获取处于共享状态的锁，它必须等待所有线程释放该锁；处于独占状态的锁阻止任何线程获取该锁，不论它们以何种方式。


## 2 Java 中的线程同步
### 2.1 单线程
任何程序至少有一个线程，即使你没有主动地创建线程，程序从一开始执行就有一个默认的线程，被称为主线程，只有一个线程的程序称为单线程程序。如下面这一简单的代码，没有显示地创建一个线程，程序从main开始执行，main本身就是一个线程(主线程)，单个线程从头执行到尾。

### 2.2 多线程
大部分情况下选择实现Runnable接口的方式会优于继承Thread的方式，因为：
1. 从 Thread 类继承会强加类层次；
2. 有些类不能继承Thread类，如要作为线程运行的类已经是某一个类的子类了，但Java只支持单继承，所以不能再继承Thread类了。

### 2.3 线程控制
#### 2.3.1 线程等待(wait、notify、notifyAll)
在Java的Object类中有三个final的方法允许线程之间进行资源对象锁的通信，他们分别是： wait(), notify() and notifyAll()。
##### wait
Object的wait方法有三个重载方法，其中一个方法wait() 是无限期(一直)等待，直到其它线程调用notify或notifyAll方法唤醒当前的线程；另外两个方法wait(long timeout) 和wait(long timeout, int nanos)允许传入 当前线程在被唤醒之前需要等待的时间，timeout为毫秒数，nanos为纳秒数。

##### notify
notify方法只唤醒一个等待（对象的）线程并使该线程开始执行。所以如果有多个线程等待一个对象，这个方法只会唤醒其中一个线程，选择哪个线程取决于操作系统对多线程管理的实现。

##### notifyAll
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

当线程执行wait()方法的时候， 会释放当前的锁，然后让出CPU，进入等待状态。
只有当notify/notifyAll()被执行的时候，才会唤醒一个或者多个处于等待状态的线程， 然后继续往下执行，直到执行完synchronized代码块的代码或者是遇到wait再次释放锁，进入等待状态。

**可以考虑一下为什么wait()必须在synchronized中**
- 每一个对象都有一个与之相对应的监视器
- 每一个监视器里面都有一个该对象的锁和一个等待队列和一个同步队列

wait()方法的语义有两个， 一是释放当前对象锁，另一个是进入阻塞队列， 可以看出， 这些操作都是与监视器相关的，当然要制定一个监视器才能完成操作。
因wait()而导致阻塞的线城市放在阻塞队列中的，因竞争失败导致的阻塞是放在同步队列中的，notify/notifyall的本质是把阻塞队列中的线程放到同步队列中。

**为什么使用锁而不是使用其他的判断条件。**
主要是Synchronized 是一个排他锁，他可以保证可见性。
wait()一般是在控制条件Condition下协同使用，如果Condition不能满足， 那么将一直等待。就算其他线程修改了Condition, 但是第三个线程可能会再次修改这个值，让线程一直被挂起。
但是wait()的使用机制要求，他的condition的设定有其他线程设定，所以为了正确工作必须正确使用内置锁。

#### 2.3.2 线程中断(interrupt)
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

#### 2.3.3 线程合并(join)
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

#### 2.3.4 优先级(Priority)

线程优先级是指获得CPU资源的优先程序。优先级高的容易获得CPU资源，优先级底的较难获得CPU资源，表现出来的情况就是优先级越高执行的时间越多。Java中通过getPriority和setPriority方法获取和设置线程的优先级。Thread类提供了三个表示优先级的常量：MIN_PRIORITY优先级最低，为1；NORM_PRIORITY是正常的优先级；为5，MAX_PRIORITY优先级最高，为10。我们创建线程对象后，如果不显示的设置优先级的话，默认为5。
```java
thread1.setPriority(Thread.MAX_PRIORITY);
thread2.setPriority(8);
```

## 3. synchronized 的用法
synchronized是Java中的关键字，是一种同步锁。它修饰的对象有以下几种：

1. 修饰一个代码块，被修饰的代码块称为同步语句块，作用于传入的实例加锁，进入同步代码前要获得当前实例的锁
2. 修饰一个方法，被修饰的方法称为同步方法，作用于当前实例加锁，进入同步代码前要获得当前实例的锁
3. 修改一个静态的方法，作用于当前类对象加锁，进入同步代码前要获得当前类对象的锁
4. 修改一个类，指定加锁对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁。


### 3.1 synchronized 修饰一个方法或代码块
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

#### 3.1.1 synchronized 作用于不同对象
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

#### 3.1.2 synchronized 和 非synchronized方法

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


#### 修饰一个静态的方法
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

<font color=#528B8B>Java语法规定，任何线程执行同步方法、同步代码块之前，必须先获取对应的监视器。并且监听器this 和 \*.class 是不同的。
this是对用方法的对象本身 class是该类本身(只有监听器相同锁才会起作用)。
当synchronized作用于静态方法时，其锁就是当前类的class对象锁。由于静态成员不专属于任何一个实例对象，是类成员，因此通过class对象锁可以控制静态 成员的并发操作。需要注意的是如果一个线程A调用一个实例对象的非static synchronized方法，而线程B需要调用这个实例对象所属类的静态 synchronized方法，是允许的，不会发生互斥现象，因为访问静态 synchronized 方法占用的锁是当前类的class对象，而访问非静态 synchronized 方法占用的锁是当前实例对象锁。</font>

**总结：**

1. 类的每个实例都有自己的对象级别锁。当一个线程访问实例对象中的synchronized同步代码块或同步方法时，该线程便获取了该实例的对象级别锁，其他线程这时如果要访问synchronized同步代码块或同步方法，便需要阻塞等待，直到前面的线程从同步代码块或方法中退出，释放掉了该对象级别锁。

2. 访问同一个类的不同实例对象中的同步代码块，不存在阻塞等待获取对象锁的问题，因为它们获取的是各自实例的对象级别锁，相互之间没有影响。

3. 持有一个对象级别锁不会阻止该线程被交换出来，也不会阻塞其他线程访问同一示例对象中的非synchronized代码。当一个线程A持有一个对象级别锁（即进入了synchronized修饰的代码块或方法中）时，线程也有可能被交换出去，此时线程B有可能获取执行该对象中代码的时间，但它只能执行非同步代码（没有用synchronized修饰），当执行到同步代码时，便会被阻塞，此时可能线程规划器又让A线程运行，A线程继续持有对象级别锁，当A线程退出同步代码时（即释放了对象级别锁），如果B线程此时再运行，便会获得该对象级别锁，从而执行synchronized中的代码。

4. 使用synchronized（obj）同步语句块，可以获取指定对象上的对象级别锁。

5. 互斥是实现同步的一种手段，临界区、互斥量和信号量都是主要的互斥实现方式。synchronized关键字经过编译后，会在同步块的前后分别形成monitorenter和monitorexit这两个字节码指令。根据虚拟机规范的要求，在执行monitorenter指令时，首先要尝试获取对象的锁，如果获得了锁，把锁的计数器加1，相应地，在执行monitorexit指令时会将锁计数器减1，当计数器为0时，锁便被释放了。由于synchronized同步块对同一个线程是可重入的，因此一个线程可以多次获得同一个对象的互斥锁，同样，要释放相应次数的该互斥锁，才能最终释放掉该锁。

<font color=#117A65>**synchronized 的另个一重要作用：内存可见性**</font>

加锁（synchronized 同步）的功能不仅仅局限于互斥行为，同时还存在另外一个重要的方面：内存可见性。我们不仅希望防止某个线程正在使用对象状态而另一个线程在同时修改该状态，而且还希望确保当一个线程修改了对象状态后，其他线程能够看到该变化。而线程的同步恰恰也能够实现这一点。

这里比较下volatile 和synchronized 变量实现内存可见性的方法二者的区别。
> - volatile 变量是一种稍弱的同步机制在访问 volatile 变量时不会执行加锁操作，因此也就不会使执行线程阻塞，因此 volatile 变量是一种比 synchronized 关键字更轻量级的同步机制。
> - 从内存可见性的角度看，写入 volatile 变量相当于退出同步代码块，而读取 volatile 变量相当于进入同步代码块。
> - 在代码中如果过度依赖 volatile 变量来控制状态的可见性，通常会比使用锁的代码更脆弱，也更难以理解。仅当 volatile 变量能简化代码的实现以及对同步策略的验证时，才应该使用它。一般来说，用同步机制会更安全些。
> - 加锁机制（即同步机制）既可以确保可见性又可以确保原子性，而 volatile 变量只能确保可见性，原因是声明为 volatile 的简单变量如果当前值与该变量以前的值相关，那么 volatile 关键字不起作用，也就是说如下的表达式都不是原子操作：count++、count = count+1。

当且仅当满足以下所有条件时，才应该使用 volatile 变量：

> - 对变量的写入操作不依赖变量的当前值，或者你能确保只有单个线程更新变量的值。
> - 该变量没有包含在具有其他变量的不变式中。

## 4. synchronized 底层实现

> 感谢[深入理解Java并发之synchronized实现原理][11]

Java 虚拟机中的同步(Synchronization)基于进入和退出管程(Monitor)对象实现， 无论是显式同步(有明确的 monitorenter 和 monitorexit 指令,即同步代码块)还是隐式同步都是如此。在 Java 语言中，同步用的最多的地方可能是被 synchronized 修饰的同步方法。同步方法 并不是由 monitorenter 和 monitorexit 指令来实现同步的，而是由方法调用指令读取运行时常量池中方法的 ACC_SYNCHRONIZED 标志来隐式实现的。

### 4.1 理解Java对象头与Monitor
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
 ```c
ObjectMonitor() {
    \_header       = NULL;
    \_count        = 0; //记录个数
    \_waiters      = 0,
    \_recursions   = 0;
    \_object       = NULL;
    \_owner        = NULL;
    \_WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet
    \_WaitSetLock  = 0 ;
    \_Responsible  = NULL ;
    \_succ         = NULL ;
    \_cxq          = NULL ;
    FreeNext      = NULL ;
    \_EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
    \_SpinFreq     = 0 ;
    \_SpinClock    = 0 ;
    OwnerIsThread = 0 ;
}

```



ObjectMonitor中有两个队列，\_WaitSet 和 \_EntryList，用来保存ObjectWaiter对象列表( 每个等待锁的线程都会被封装成ObjectWaiter对象)，\_owner指向持有ObjectMonitor对象的线程，当多个线程同时访问一段同步代码时，首先会进入 \_EntryList 集合，当线程获取到对象的monitor 后进入 \_Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1，若线程调用 wait() 方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSe t集合中等待被唤醒。若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)。

### 4.2 synchronized代码块底层原理
​```Java
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

### 4.3 synchronized方法底层原理

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

### 4.4 synchronized底层原理
其应用层的语义是可以把任何一个非null对象 作为"锁"，当synchronized作用在方法上时，锁住的便是对象实例（this）；当作用在静态方法时锁住的便是对象对应的Class实例，因为 Class数据存在于永久带，因此静态方法锁相当于该类的一个全局锁；当synchronized作用于某一个对象实例时，锁住的便是对应的代码块。在 HotSpot JVM实现中，锁有个专门的名字：对象监视器。

>在java虚拟机中，每个对象和类在逻辑上都是和一个监视器相关联的。
> - 对于对象来说，相关联的监视器保护对象的实例变量。
> - 对于类来说，监视器保护类的类变量。
#### 4.4.1 线程状态及状态转换
当多个线程同时请求某个对象监视器时，对象监视器会设置几种状态用来区分请求的线程：

1. Contention List：所有请求锁的线程将被首先放置到该竞争队列
2. Entry List：Contention List中那些有资格成为候选人的线程被移到Entry List
3. Wait Set：那些调用wait方法被阻塞的线程被放置到Wait Set
4. OnDeck：任何时刻最多只能有一个线程正在竞争锁，该线程称为OnDeck
5. Owner：获得锁的线程称为Owner
6. !Owner：释放锁的线程

![img][19]

新请求锁的线程将首先被加入到ConetentionList中，当某个拥有锁的线程（Owner状态）调用unlock之后，如果发现 EntryList为空则从ContentionList中移动线程到EntryList，下面说明下ContentionList和EntryList 的实现方式。

#### 4.4.2 ContentionList 虚拟队列
ContentionList并不是一个真正的Queue，而只是一个虚拟队列，原因在于ContentionList是由Node及其next指 针逻辑构成，并不存在一个Queue的数据结构。ContentionList是一个后进先出（LIFO）的队列，每次新加入Node时都会在队头进行， 通过CAS改变第一个节点的的指针为新增节点，同时设置新增节点的next指向后续节点，而取得操作则发生在队尾。显然，该结构其实是个Lock- Free的队列。

因为只有Owner线程才能从队尾取元素，也即线程出列操作无争用，当然也就避免了CAS的ABA问题。

![img][20]

#### 4.4.3 EntryList

EntryList与ContentionList逻辑上同属等待队列，ContentionList会被线程并发访问，为了降低对 ContentionList队尾的争用，而建立EntryList。Owner线程在unlock时会从ContentionList中迁移线程到 EntryList，并会指定EntryList中的某个线程（一般为Head）为Ready（OnDeck）线程。Owner线程并不是把锁传递给 OnDeck线程，只是把竞争锁的权利交给OnDeck，OnDeck线程需要重新竞争锁。这样做虽然牺牲了一定的公平性，但极大的提高了整体吞吐量，在 Hotspot中把OnDeck的选择行为称之为“竞争切换”。

OnDeck线程获得锁后即变为owner线程，无法获得锁则会依然留在EntryList中，考虑到公平性，在EntryList中的位置不 发生变化（依然在队头）。如果Owner线程被wait方法阻塞，则转移到WaitSet队列；如果在某个时刻被notify/notifyAll唤醒， 则再次转移到EntryList。

#### 4.4.4 自旋锁

那些处于ContetionList、EntryList、WaitSet中的线程均处于阻塞状态，阻塞操作由操作系统完成（在Linxu下通 过pthread_mutex_lock函数）。线程被阻塞后便进入内核（Linux）调度状态，这个会导致系统在用户态与内核态之间来回切换，严重影响 锁的性能

缓解上述问题的办法便是自旋，其原理是：当发生争用时，若Owner线程能在很短的时间内释放锁，则那些正在争用线程可以稍微等一等（自旋）， 在Owner线程释放锁后，争用线程可能会立即得到锁，从而避免了系统阻塞。但Owner运行的时间可能会超出了临界值，争用线程自旋一段时间后还是无法 获得锁，这时争用线程则会停止自旋进入阻塞状态（后退）。基本思路就是自旋，不成功再阻塞，尽量降低阻塞的可能性，这对那些执行时间很短的代码块来说有非 常重要的性能提高。自旋锁有个更贴切的名字：自旋-指数后退锁，也即复合锁。很显然，自旋在多处理器上才有意义。

还有个问题是，线程自旋时做些啥？其实啥都不做，可以执行几次for循环，可以执行几条空的汇编指令，目的是占着CPU不放，等待获取锁的机 会。所以说，自旋是把双刃剑，如果旋的时间过长会影响整体性能，时间过短又达不到延迟阻塞的目的。显然，自旋的周期选择显得非常重要，但这与操作系统、硬 件体系、系统的负载等诸多场景相关，很难选择，如果选择不当，不但性能得不到提高，可能还会下降，因此大家普遍认为自旋锁不具有扩展性。

自旋优化策略

对自旋锁周期的选择上，HotSpot认为最佳时间应是一个线程上下文切换的时间，但目前并没有做到。经过调查，目前只是通过汇编暂停了几个CPU周期，除了自旋周期选择，HotSpot还进行许多其他的自旋优化策略，具体如下：

如果平均负载小于CPUs则一直自旋

如果有超过(CPUs/2)个线程正在自旋，则后来线程直接阻塞

如果正在自旋的线程发现Owner发生了变化则延迟自旋时间（自旋计数）或进入阻塞

如果CPU处于节电模式则停止自旋

自旋时间的最坏情况是CPU的存储延迟（CPU A存储了一个数据，到CPU B得知这个数据直接的时间差）

自旋时会适当放弃线程优先级之间的差异

那synchronized实现何时使用了自旋锁？答案是在线程进入ContentionList时，也即第一步操作前。线程在进入等待队列时 首先进行自旋尝试获得锁，如果不成功再进入等待队列。这对那些已经在等待队列中的线程来说，稍微显得不公平。还有一个不公平的地方是自旋线程可能会抢占了 Ready线程的锁。自旋锁由每个监视对象维护，每个监视对象一个。

#### 4.4.5 JVM1.6偏向锁

在JVM1.6中引入了偏向锁，偏向锁主要解决无竞争下的锁性能问题，首先我们看下无竞争下锁存在什么问题：

现在几乎所有的锁都是可重入的，也即已经获得锁的线程可以多次锁住/解锁监视对象，按照之前的HotSpot设计，每次加锁/解锁都会涉及到一些CAS操 作（比如对等待队列的CAS操作），CAS操作会延迟本地调用，因此偏向锁的想法是一旦线程第一次获得了监视对象，之后让监视对象“偏向”这个 线程，之后的多次调用则可以避免CAS操作，说白了就是置个变量，如果发现为true则无需再走各种加锁/解锁流程。但还有很多概念需要解释、很多引入的 问题需要解决：

### 4.5 CAS及SMP架构
CAS为什么会引入本地延迟？这要从SMP（对称多处理器）架构说起

![img][21]

其意思是所有的CPU会共享一条系统总线（BUS），靠此总线连接主存。每个核都有自己的一级缓存，各核相对于BUS对称分布，因此这种结构称为“对称多处理器”。

而CAS的全称为Compare-And-Swap，是一条CPU的原子指令，其作用是让CPU比较后原子地更新某个位置的值，经过调查发现， 其实现方式是基于硬件平台的汇编指令，就是说CAS是靠硬件实现的，JVM只是封装了汇编调用，那些AtomicInteger类便是使用了这些封装后的 接口。

Core1和Core2可能会同时把主存中某个位置的值Load到自己的L1 Cache中，当Core1在自己的L1 Cache中修改这个位置的值时，会通过总线，使Core2中L1 Cache对应的值“失效”，而Core2一旦发现自己L1 Cache中的值失效（称为Cache命中缺失）则会通过总线从内存中加载该地址最新的值，大家通过总线的来回通信称为“Cache一致性流量”，因为总 线被设计为固定的“通信能力”，如果Cache一致性流量过大，总线将成为瓶颈。而当Core1和Core2中的值再次一致时，称为“Cache一致 性”，从这个层面来说，锁设计的终极目标便是减少Cache一致性流量。

而CAS恰好会导致Cache一致性流量，如果有很多线程都共享同一个对象，当某个Core CAS成功时必然会引起总线风暴，这就是所谓的本地延迟，本质上偏向锁就是为了消除CAS，降低Cache一致性流量。

**Cache一致性：**
上面提到Cache一致性，其实是有协议支持的，现在通用的协议是MESI（最早由Intel开始支持），具体参考：http://en.wikipedia.org/wiki/MESI_protocol

**Cache一致性流量的例外情况：**
其实也不是所有的CAS都会导致总线风暴，这跟Cache一致性协议有关，具体参考：http://blogs.oracle.com/dave/entry/biased_locking_in_hotspot

NUMA(Non Uniform Memory Access Achitecture）架构：
与SMP对应还有非对称多处理器架构，现在主要应用在一些高端处理器上，主要特点是没有总线，没有公用主存，每个Core有自己的内存，针对这种结构此处不做讨论。

3.2 偏向解除

偏向锁引入的一个重要问题是，在多争用的场景下，如果另外一个线程争用偏向对象，拥有者需要释放偏向锁，而释放的过程会带来一些性能开销，但总体说来偏向锁带来的好处还是大于CAS代价的。

关于锁，JVM中还引入了一些其他技术比如锁膨胀等，这些与自旋锁、偏向锁相比影响不是很大，这里就不做介绍。
通过上面的介绍可以看出，synchronized的底层实现主要依靠Lock-Free的队列，基本思路是自旋后阻塞，竞争切换后继续竞争锁，稍微牺牲了公平性，但获得了高吞吐量。

## 5 Java虚拟机对synchronized的优化
锁的状态总共有四种，无锁状态、偏向锁、轻量级锁和重量级锁。随着锁的竞争，锁可以从偏向锁升级到轻量级锁，再升级的重量级锁，但是锁的升级是单向的，也就是说只能从低到高升级，不会出现锁的降级。

### 5.1 偏向锁
偏向锁是Java 6之后加入的新锁，它是一种针对加锁操作的优化手段，经过研究发现，在大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，因此为了减少同一线程获取锁(会涉及到一些CAS操作,耗时)的代价而引入偏向锁。偏向锁的核心思想是，如果一个线程获得了锁，那么锁就进入偏向模式，此时Mark Word 的结构也变为偏向锁结构，当这个线程再次请求锁时，无需再做任何同步操作，即获取锁的过程，这样就省去了大量有关锁申请的操作，从而也就提供程序的性能。所以，对于没有锁竞争的场合，偏向锁有很好的优化效果，毕竟极有可能连续多次是同一个线程申请相同的锁。但是对于锁竞争比较激烈的场合，偏向锁就失效了，因为这样场合极有可能每次申请锁的线程都是不相同的，因此这种场合下不应该使用偏向锁，否则会得不偿失，需要注意的是，偏向锁失败后，并不会立即膨胀为重量级锁，而是先升级为轻量级锁。下面我们接着了解轻量级锁。

### 5.2 轻量级锁
倘若偏向锁失败，虚拟机并不会立即升级为重量级锁，它还会尝试使用一种称为轻量级锁的优化手段(1.6之后加入的)，此时Mark Word 的结构也变为轻量级锁的结构。轻量级锁能够提升程序性能的依据是“对绝大部分的锁，在整个同步周期内都不存在竞争”，注意这是经验数据。需要了解的是，轻量级锁所适应的场景是线程交替执行同步块的场合，如果存在同一时间访问同一锁的场合，就会导致轻量级锁膨胀为重量级锁。

### 5.3 自旋锁
轻量级锁失败后，虚拟机为了避免线程真实地在操作系统层面挂起，还会进行一项称为自旋锁的优化手段。这是基于在大多数情况下，线程持有锁的时间都不会太长，如果直接挂起操作系统层面的线程可能会得不偿失，毕竟操作系统实现线程之间的切换时需要从用户态转换到核心态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，因此自旋锁会假设在不久将来，当前的线程可以获得锁，因此虚拟机会让当前想要获取锁的线程做几个空循环(这也是称为自旋的原因)，一般不会太久，可能是50个循环或100循环，在经过若干次循环后，如果得到锁，就顺利进入临界区。如果还不能获得锁，那就会将线程在操作系统层面挂起，这就是自旋锁的优化方式，这种方式确实也是可以提升效率的。最后没办法也就只能升级为重量级锁了。

### 5.4 锁消除
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

## 6 synchronized的几个特点
### 6.1 synchronized的可重入性
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

### 6.2 等待唤醒机制与synchronized
所谓等待唤醒机制本篇主要指的是`notify/notifyAll`和`wait`方法，在使用这3个方法时，必须处于`synchronized`代码块或者`synchronized`方法中，否则就会抛出`IllegalMonitorStateException`异常，这是因为调用这几个方法前必须拿到当前对象的监视器`monitor`对象，也就是说`notify/notifyAll`和`wait`方法依赖于`monitor`对象，在前面的分析中，我们知道`monitor` 存在于对象头的`Mark Word` 中(存储`monitor`引用指针)，而`synchronized`关键字可以获取 `monitor` ，这也就是为什么`notify/notifyAll`和`wait`方法必须在`synchronized`代码块或者`synchronized`方法调用的原因。

需要特别理解的一点是，与`sleep`方法不同的是`wait`方法调用完成后，线程将被暂停，但`wait`方法将会释放当前持有的监视器锁(monitor)，直到有线程调用`notify/notifyAll`方法后方能继续执行，而`sleep`方法只让线程休眠并不释放锁。同时`notify/notifyAll`方法调用后，并不会马上释放监视器锁，而是在相应的`synchronized(){}/synchronized`方法执行结束后才自动释放锁。

### 6.3 线程中断与synchronized
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
### 6.4 synchronized 与锁等待
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

[synchronized锁分析][13]

[Java SE 1.6对synchronized的优化][14]

[ReenTrantLock可重入锁（和synchronized的区别）总结][15]

[ReentrantLock源码之一lock方法解析(锁的获取)][16]

[jvm对内置锁的优化][17]

[Java中synchronized的实现原理与应用][18]

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
[13]:https://blog.csdn.net/xad707348125/article/details/46956911
[14]:https://blog.csdn.net/limengliang4007/article/details/77513256
[15]:https://blog.csdn.net/qq838642798/article/details/65441415
[16]:http://www.blogjava.net/zhanglongsr/articles/356782.html
[17]:https://blog.csdn.net/maozhr720/article/details/76518175
[18]:https://blog.csdn.net/u012465296/article/details/53022317
[19]:http://static.open-open.com/lib/uploadImg/20121109/20121109112521_220.jpg
[20]:http://static.open-open.com/lib/uploadImg/20121109/20121109112521_981.jpg
[21]:http://static.open-open.com/lib/uploadImg/20121109/20121109112521_777.jpg
