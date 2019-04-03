

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

### 关于内存泄漏
实际上，为了解决threadLocal潜在的内存泄漏的问题已经做了一些改进。在threadLocal的set和get方法中都有相应的处理。

在`ThreadLocalMap.set()`方法中已经做了一些处理:
```Java
if (k == null) {
    replaceStaleEntry(key, value, i);
    return;
}

...

if (!cleanSomeSlots(i, sz) && sz >= threshold)
    rehash();
```
- 如果当前table[i]!=null的话说明hash冲突就需要向后环形查找，若在查找过程中遇到脏entry就通过replaceStaleEntry进行处理
- 如果当前table[i]==null的话说明新的entry可以直接插入，但是插入后会调用cleanSomeSlots方法检测并清除脏entry






### 关于到底能不能set之前get
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


```Java

```

```Java

```

```Java

```

```Java

```

```Java

```

```Java

```
























[1]:https://www.jianshu.com/p/30ee77732843
[2]:https://upload-images.jianshu.io/upload_images/2615789-12aef2e6ff040cae.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/714/format/webp
[3]:https://upload-images.jianshu.io/upload_images/2615789-32b422909f2f933c.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/696/format/webp
[4]:https://upload-images.jianshu.io/upload_images/2615789-0d85565e94c4bd6b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/722/format/webp


/
