

# ThreadLocal


## 背景
ThreadLocal，很多地方叫做线程本地变量，也有些地方叫做线程本地存储，其实意思差不多。可能很多朋友都知道ThreadLocal为变量在每个线程中都创建了一个副本，那么每个线程可以访问自己内部的副本变量。

同步机制以“时间换空间”，由于每个线程在同一时刻共享对象只能被一个线程访问造成整体上响应时间增加，但是对象只占有一份内存，牺牲了时间效率换来了空间效率即“时间换空间”。而threadLocal，为每个线程都分配了一份对象，自然而然内存使用率增加，每个线程各用各的，整体上时间效率要增加很多，牺牲了空间效率换来时间效率即“空间换时间”。


首先我们可以看一个完整的例子:
```java
public class ThreadLocalTest {

    public Integer getAfterSet() {
        ThreadLocal<Integer> threadLocal = new ThreadLocal<>();
        threadLocal.set(1);
        return threadLocal.get();
    }

    public Integer getWithoutSet() {
        ThreadLocal<Integer> threadLocal = new ThreadLocal<>();
        return threadLocal.get();
    }

    @Test
    public void test() {
        Assert.assertEquals(1, getAfterSet().intValue());
        Assert.assertNull(getWithoutSet());
    }
}
```

## 1. ThreadLocal 深入源码分析
ThreadLocal 被设计成一个，可以在同一个线程中存在多个ThreadLocal对象.
所以就存在了一个ThreadLocalMap, 用于维护相同线程中，多个Threadlocal实例。

对应上面的代码，我们一步一步的分析ThreadLocal的调用过程，首先分析`threadLocal.set(1);`

### 1.1 初始化和插入数据 `threadLocal.set(1)`
我们可以首先根据源码来分析一下:
```Java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);     --------- 1
    if (map != null)
        map.set(this, value);           --------- 3
    else
        createMap(t, value);            --------- 2
}
```
#### 1.1.1 执行步骤
这个方法引出以下几个步骤:
1. 通过当前线程去获得`ThreadLocalMap`
2. 如果`ThreadLocalMap`实例不存在，就创建一个map， 并把value插入到map中
3. 如果`ThreadLocalMap`实例存在，就把value插入到map中

#### 1.1.2 获得`ThreadLocalMap`

我们可以通过方法getMap看出，`ThreadLocalMap`实例实际上是绑定在Thread类中的。

```Java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

```Java
public class Thread implements Runnable {
  /* ThreadLocal values pertaining to this thread. This map is maintained
   * by the ThreadLocal class.
   */
  ThreadLocal.ThreadLocalMap threadLocals = null;
...
}
```

而且在`Thread`类中，默认`ThreadLocalMap`是`null`， 只有当线程使用了`ThreadLocal`的时候，才会真正的创建`ThreadLocalMap`对象。

#### 1.1.3 创建`ThreadLocalMap`过程

```Java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
```Java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```
我们一步一步地看待这段代码:
1. `ThreadLocalMap`维护了一个`Entry`数组实例
2. `ThreadLocalMap`使用`ThreadLocal`作为键值, `ThreadLocal`中的`hash`值是`ThreadLocal`中的一个属性
3. 通过`hash`值在被`hash`之后，插入对应的位置

由此可以看出， `threadLocalHashCode`是一个递增的数字，每当生成一个新的`ThreadLocal`实例，就会使`threadLocalHashCode`递增加一。
```Java
public class ThreadLocal<T> {
  private final int threadLocalHashCode = nextHashCode();
  private static AtomicInteger nextHashCode = new AtomicInteger();
  private static final int HASH_INCREMENT = 0x61c88647;

 private static int nextHashCode() {
     return nextHashCode.getAndAdd(HASH_INCREMENT);
 }
}
```

现在来对`set`方法进行总结一下：

通过当前线程对象`thread`获取该`thread`所维护的`threadLocalMap`,若`threadLocalMap`不为`null`,则以`threadLocal`实例为`key`,值为`value`的键值对存入`threadLocalMap`,若`threadLocalMap`为`null`的话，就新建`threadLocalMap`然后在以`threadLocal`为键，值为`value`的键值对存入即可。


#### 1.1.4 插入数据到ThreadLocalMap
首先看源码怎么写:
```Java
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);   //注意这里，很搞事情.
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

其实这里面的`Hash`表的算法更像我们在课堂上学到的`hash`算法，如果当前的`hash`所指示的数组所在地方已经被占用，就沿着当前表往下遍历，一直遍历到第一个为`null`的数组下标，然后把要插入的值插入到`ThreadLocalMap`。

注意：**不会发生数组所有的数据都被插入了，这里面的`threshold`会控制到达`2/3`之后就会扩容**

接下来，我们可以看看`ThreadLocal.get()`是如何工作的。

### 1.2 获取线程副本`threadLocal.get()`
```Java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```
和`set`相对应，我们在`get`的时候，通过`map`来查询结果，有可能`map`并不存在，也就是说没有执行`set`，就直接`get`。

```Java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
protected T initialValue() {
    return null;
}
```
这个方法比较简单，就是初始化一个`map`，然后把空值传入进去.
这里面就涉及了完整的一个`map`的创建过程然后被`Thread`实例引用。


> 其实到这里，一个完整的`ThreadLocal`是如何为每个线程创建变量的副本的过程就已经很清晰了.
>
> 首先，在每个线程`Thread`内部有一个`ThreadLocal.ThreadLocalMap`类型的成员变量`threadLocals`，这个`threadLocals`就是用来存储实际的变量副本的，键值为当前`ThreadLocal`变量，`value`为变量副本（即T类型的变量）。
>
> 初始时，在`Thread`里面，`threadLocals`为空，当通过`ThreadLocal`变量调用`get()`方法或者`set()`方法，就会对`Thread`类中的`threadLocals`进行初始化，并且以当前`ThreadLocal`变量为键值，以`ThreadLocal`要保存的副本变量为`value`，存到`threadLocals`。
>
> 然后在当前线程里面，如果要使用副本变量，就可以通过`get`方法在`threadLocals`里面查找。

## 2. ThreadLocalMap详解
> 此处摘录自[这里][1], 如果想更深入理解，请前往阅读.

从上面的分析我们已经知道，数据其实都放在了`threadLocalMap`中，`threadLocal`的`get`，`set`和`remove`方法实际上具体是通过`threadLocalMap`的`getEntry`,`set`和`remove`方法实现的。如果想真正全方位的弄懂`threadLocal`，势必得在对`threadLocalMap`做一番理解。

### 2.1 Entry数据结构

Entry是一个以ThreadLocal为key,Object为value的键值对，另外需要注意的是这里的threadLocal是弱引用，因为Entry继承了WeakReference，在Entry的构造方法中，调用了super(k)方法就会将threadLocal实例包装成一个WeakReferenece。

![ThreadLocal引用关系][2]

> 注意上图中的实线表示强引用，虚线表示弱引用。

每个线程实例中可以通过`threadLocals`获取到`threadLocalMap`，而`threadLocalMap`实际上就是一个以`threadLocal`实例为`key`，任意对象为`value`的`Entry`数组。当我们为`threadLocal`变量赋值，实际上就是以当前`threadLocal`实例为`key`，值为`value`的`Entry`往这个`threadLocalMap`中存放。需要注意的是`Entry`中的`key`是弱引用，当`threadLocal`外部强引用被置为`null(threadLocalInstance=null)`,那么系统`GC`的时候，根据可达性分析，这个`threadLocal`实例就没有任何一条链路能够引用到它，这个`ThreadLocal`势必会被回收，这样一来，`ThreadLocalMap`中就会出现`key`为`null`的`Entry`，就没有办法访问这些 `key` 为`null`的`Entry`的`value`，如果当前线程再迟迟不结束的话，这些`key`为`null`的`Entry`的`value`就会一直存在一条强引用链：`Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value`永远无法回收，造成内存泄漏。

### 2.2 散列表

在实际使用中，经常会出现多个关键字散列值相同的情况（被映射到数组的同一个位置），我们将这种情况称为散列冲突。为了解决散列冲突，主要采用下面两种方式： `分离链表法（separate chaining）`和`开放定址法（open addressing）`

#### 2.2.1 分离链表法
分散链表法使用链表解决冲突，将散列值相同的元素都保存到一个链表中。当查询的时候，首先找到元素所在的链表，然后遍历链表查找对应的元素，典型实现为hashMap，concurrentHashMap的拉链法。

![分离链表法示意图][3]

#### 2.2.2 开放定址法
开放定址法不会创建链表，当关键字散列到的数组单元已经被另外一个关键字占用的时候，就会尝试在数组中寻找其他的单元，直到找到一个空的单元。探测数组空单元的方式有很多，这里介绍一种最简单的 -- 线性探测法。线性探测法就是从冲突的数组单元开始，依次往后搜索空单元，如果到数组尾部，再从头开始搜索（环形查找）。

![开放定址法示意图][4]
之所以采用不同的方式主要是因为：在 ThreadLocalMap 中的散列值分散的十分均匀，很少会出现冲突。并且 ThreadLocalMap 经常需要清除无用的对象，使用纯数组更加方便。



## ThreadLocal的坑

## 1. 内存泄露

涉及三个方法;


#### `expungeStaleEntry(int staleSlot)`
```Java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    //清除当前脏entry
    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
        //2.往后环形继续查找,直到遇到table[i]==null时结束
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
      //3. 如果在向后搜索过程中再次遇到脏entry，同样将其清理掉
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
          //处理rehash的情况
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}

```
清理掉当前脏entry后，并没有闲下来继续向后搜索，若再次遇到脏entry继续将其清理，直到哈希桶（table[i]）为null时退出。因此方法执行完的结果为 从当前脏entry（staleSlot）位到返回的i位，这中间所有的entry不是脏entry。为什么是遇到null退出呢？原因是存在脏entry的前提条件是 当前哈希桶（table[i]）不为null,只是该entry的key域为null。如果遇到哈希桶为null,很显然它连成为脏entry的前提条件都不具备。

#### `cleanSomeSlots(int i, int n)`

```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}

```
这个源码很简单， 就是向后循环查询，主要用于扫描控制（scan control），从while中是通过n来进行条件判断的说明n就是用来控制扫描趟数（循环次数）的。在扫描过程中，如果没有遇到脏entry就整个扫描过程持续log2(n)次，log2(n)的得来是因为n >>>= 1，每次n右移一位相当于n除以2。如果在扫描过程中遇到脏entry的话就会令n为当前hash表的长度（n=len），再扫描log2(n)趟，注意此时n增加无非就是多增加了循环次数从而通过nextIndex往后搜索的范围扩大，示意图如下

![cleanSomeSlots示意图][7]

现在对cleanSomeSlot方法做一下总结:

- 从当前位置i处（位于i处的entry一定不是脏entry）为起点在初始小范围（log2(n)，n为哈希表已插入entry的个数size）开始向后搜索脏entry，若在整个搜索过程没有脏entry，方法结束退出
- 如果在搜索过程中遇到脏entryt通过expungeStaleEntry方法清理掉当前脏entry，并且该方法会返回下一个哈希桶(table[i])为null的索引位置为i。这时重新令搜索起点为索引位置i，n为哈希表的长度len，再次扩大搜索范围为log2(n')继续搜索。



下面，以一个例子更清晰的来说一下，假设当前table数组的情况如下图。

> 这一段例子引用自这里[这里][8]

![][9]

- 如图当前n等于hash表的size即n=10，i=1,在第一趟搜索过程中通过nextIndex,i指向了索引为2的位置，此时table[2]为null，说明第一趟未发现脏entry,则第一趟结束进行第二趟的搜索。

- 第二趟所搜先通过nextIndex方法，索引由2的位置变成了i=3,当前table[3]!=null但是该entry的key为null，说明找到了一个脏entry，先将n置为哈希表的长度len,然后继续调用expungeStaleEntry方法，该方法会将当前索引为3的脏entry给清除掉（令value为null，并且table[3]也为null）,但是该方法可不想偷懒，它会继续往后环形搜索，往后会发现索引为4,5的位置的entry同样为脏entry，索引为6的位置的entry不是脏entry保持不变，直至i=7的时候此处table[7]位null，该方法就以i=7返回。至此，第二趟搜索结束；

- 由于在第二趟搜索中发现脏entry，n增大为数组的长度len，因此扩大搜索范围（增大循环次数）继续向后环形搜索；

- 直到在整个搜索范围里都未发现脏entry，cleanSomeSlot方法执行结束退出。


#### `replaceStaleEntry`

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    // Back up to check for prior stale entry in current run.
    // We clean out whole runs at a time to avoid continual
    // incremental rehashing due to garbage collector freeing
    // up refs in bunches (i.e., whenever the collector runs).
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

    // Find either the key or trailing null slot of run, whichever
    // occurs first
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // If we find key, then we need to swap it
        // with the stale entry to maintain hash table order.
        // The newly stale slot, or any other stale slot
        // encountered above it, can then be sent to expungeStaleEntry
        // to remove or rehash all of the other entries in run.
        if (k == key) {
          //如果在向后环形查找过程中发现key相同的entry就覆盖并且和脏entry进行交换
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // Start expunge at preceding stale entry if it exists
            //如果在前面的向前查找过程中还未发现脏entry，那么就以当前位置作为cleanSomeSlots
            //的起点
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // If we didn't find stale entry on backward scan, the
        // first stale entry seen while scanning for key is the
        // first still present in the run.
        //与上面相同， 如果slotToExpunge  不同，就可以认定， 前面改过， 肯定比当前的要早
        //就不要替换， 如果相同，就说明前面没有改过，就替换掉.
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // If key not found, put new entry in stale slot
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // If there are any other stale entries in run, expunge them
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

我们细致的梳理一下，这个函数做了什么:

0. 我们可以认定入参 staleSlot所指向的entry是一个脏entry, 这是必然的.

1. 首先，向前查询，查验,在当前脏entry之前，是不是还存在脏entry, 当然， 不会无限查询，只会查询不是null的.
   如果存在多个，就返回slot最小的那个.
   记录下来这个值

2. 向后查询， 还是以null结束.

3. 如果在循环查询的过程中, 发现了要插入的key已经存在
   - 和staleslot对调，并且调整当前值. 然后删掉那个脏entry
   - 然后执行`cleanSomeSlots(expungeStaleEntry(slotToExpunge), len)` 返回

4. 如果向后查询的过程中，key不存在
   - 把要插入的key-value， 插入staleSlot所在的entry.
   - 然后再次执行`cleanSomeSlots(expungeStaleEntry(slotToExpunge), len)`

其实也可以使用一些图标来形象地展示:

分为四种情况:

**前向有脏entry 后向环形查找找到可覆盖的entry**

![向前环形搜索到脏entry，向后环形查找到可覆盖的entry的情况][10]

这里，slotToExpunge的指向就是前面的脏entry.




**前向有脏entry 后向环形查找未找到可覆盖的entry**

![向前环形搜索到脏entry，后向环形查找未找到可覆盖的entry][11]

这里，slotToExpunge的指向就是前面的脏entry.


**前向没有脏entry 后向环形查找找到可覆盖的entry**

![前向没有脏entry，后向环形查找找到可覆盖的entry][12]

这里，slotToExpunge的指向的就是向后循环所查找到的第一个脏entry.


**前向没有脏entry 2.2后向环形查找未找到可覆盖的entry**

![前向没有脏entry，后向环形查找未找到可覆盖的entry][13]

这里，slotToExpunge的指向的就是向后循环所查找到的第一个脏entry.


### 为什么使用弱引用？
从文章开头通过threadLocal,threadLocalMap,entry的引用关系看起来threadLocal存在内存泄漏的问题似乎是因为threadLocal是被弱引用修饰的。

假设threadLocal使用的是强引用，在业务代码中执行threadLocalInstance==null操作，以清理掉threadLocal实例的目的，但是因为threadLocalMap的Entry强引用threadLocal，因此在gc的时候进行可达性分析，threadLocal依然可达，对threadLocal并不会进行垃圾回收，这样就无法真正达到业务逻辑的目的，出现逻辑错误


假设Entry弱引用threadLocal，尽管会出现内存泄漏的问题，但是在threadLocal的生命周期里（set,getEntry,remove）里，都会针对key为null的脏entry进行处理。
从以上的分析可以看出，使用弱引用的话在threadLocal生命周期里会尽可能的保证不出现内存泄漏的问题，达到安全的状态。



## 2. 关于到底能不能set之前get
有的博客会说`get`之前必须`set`，否则会报空指针异常. **这个理解是错误的**,正如前面的例子，并不会报错，只是会返回一个`null`，读完代码，也可以发现这一点。

我们以这个代码为例，这里是不会报错的。
```java
public class ThreadLocalTest {

    public Integer getAfterSet() {
        ThreadLocal<Integer> threadLocal = new ThreadLocal<>();
        threadLocal.set(1);
        return threadLocal.get();
    }

    public Integer getWithoutSet() {
        ThreadLocal<Integer> threadLocal = new ThreadLocal<>();
        return threadLocal.get();
    }

    @Test
    public void test() {
        Assert.assertEquals(1, getAfterSet().intValue());
        Assert.assertNull(getWithoutSet());
    }
}
```

但是如果代码改成这个样子， 把函数返回从包装类型变成基本数据类型，才会真正的报出`NullPointerException`异常。

```java
public class ThreadLocalTest {

    public Integer getAfterSet() {
        ThreadLocal<Integer> threadLocal = new ThreadLocal<>();
        threadLocal.set(1);
        return threadLocal.get();
    }

    public int getWithoutSet() {
        ThreadLocal<Integer> threadLocal = new ThreadLocal<>();
        return threadLocal.get();
    }

    @Test
    public void test() {
        Assert.assertEquals(1, getAfterSet().intValue());
        Assert.assertEquals(0, getWithoutSet());
    }
}
```

我们把代码反编译了之后，会发现

包装数据类型变成了以下字节码

```Java
public Integer getLong() {
    System.out.println(this.longLocal);
    return (Integer)this.longLocal.get();
}
```

简单数据类型变成了以下字节码
```Java
public int getLong() {
    System.out.println(this.longLocal);
    return (int)this.longLocal.get();
}
```

这个问题就已经不再属于`ThreadLocal`的范畴了，属于强制类型装换是怎么做的范畴了.
对`null`， 他们都是怎么做的:
首先，`Java`允许`(Object)null`的存在，只是这样做，依旧是返回`null`， 也就是没有任何的引用。
但是，`Java`不允许`(int)null`的存在，因为，`int`如果没有被赋值，会返回一个默认值，但是，`null`没有默认值，所以会在这里报错。
而不是什么`ThreadLocal`不允许先`get`后`set`的操作， 请注意。


[1]:https://www.jianshu.com/p/30ee77732843
[2]:https://upload-images.jianshu.io/upload_images/2615789-12aef2e6ff040cae.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/714/format/webp
[3]:https://upload-images.jianshu.io/upload_images/2615789-32b422909f2f933c.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/696/format/webp
[4]:https://upload-images.jianshu.io/upload_images/2615789-0d85565e94c4bd6b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/722/format/webp

[7]:https://upload-images.jianshu.io/upload_images/2615789-176285739b74da18.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/543/format/webp
[8]:https://www.jianshu.com/p/dde92ec37bd1
[9]: https://upload-images.jianshu.io/upload_images/2615789-217512cee7e45fc7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/737/format/webp
[10]: https://upload-images.jianshu.io/upload_images/2615789-ebc60645134a0342.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/744/format/webp
[11]: https://upload-images.jianshu.io/upload_images/2615789-423c8c8dfb2e9557.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/746/format/webp
[12]: https://upload-images.jianshu.io/upload_images/2615789-018d077773a019dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/871/format/webp
[13]: https://upload-images.jianshu.io/upload_images/2615789-eee96f3eca481ae0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/816/format/webp
