# 泛型

## 1. 泛型的定义
泛型，即“参数化类型”。一提到参数，最熟悉的就是定义方法时有形参，然后调用此方法时传递实参。那么参数化类型怎么理解呢？顾名思义，就是将类型由原来的具体的类型参数化，类似于方法中的变量参数，此时类型也定义成参数形式（可以称之为类型形参），然后在使用/调用时传入具体的类型（类型实参）。

泛型的本质是为了参数化类型（在不创建新的类型的情况下，通过泛型指定的不同类型来控制形参具体限制的类型）。也就是说在泛型使用过程中，操作的数据类型被指定为一个参数，这种参数类型可以用在类、接口和方法中，分别被称为泛型类、泛型接口、泛型方法。

## 2. 概述
### 2.1 解决的问题
在开始介绍泛型的知识之前，先来了解一下泛型解决了什么问题，换句话说，为什么需要使用泛型。集合类是在开发过程中经常要用的类，下面是不使用泛型的代码示例。
```Java
// 定义一个List，add()可以存放Object及其子类实例
List list = new ArrayList();
list.add(123); // 合法
list.add("123"); // 合法

// 我们在编译时无法知晓list到底存放的什么数据，于是在进行强制转换时发生异常
int i = (Integer) list.get(1); // 抛出ClassCastException异常
```
上面的代码首先实例化一个`ArrayList`对象，它可以存放所有Object及其子类实例。分别`add`一个`Integer`类型对象和`String`类型对象，我们原本以为`list`中存放的全部是`Integer`类型对象，于是在使用`get()`方法获取对象后进行强制转换。从代码中可以看到，索引值为1的位置放置的`String`类型，很显然在进行强制转换时会抛出`ClassCastException`（类型转换异常）。由于这种异常只会发生在运行时，我们在开发时稍有不慎，就会直接掉到坑里，还很难排查出问题。

为什么会出现这种问题呢？

1. 集合本身无法对其存放的对象类型进行限定，可以涵盖Java中的所有类型。缺口太大，导致各种蛇、蚁、虫、鼠通通都可以进来。
2. 由于我们要使用的实际存放类型的方法，所以不可避免地要进行类型转换。小对象转大对象很容易，大对象转小对象则有很大的风险，因为在编译时，我们无从得知对象真正的类型。

泛型就是为了解决这类问题而诞生的。

## 3. 特性
在编译之后程序会采取去泛型化的措施。也就是说Java中的泛型，只在编译阶段有效。在编译过程中，正确检验泛型结果后，会将泛型的相关信息擦出，并且在对象进入和离开方法的边界处添加类型检查和类型转换的方法。也就是说，泛型信息不会进入到运行时阶段。

**对此总结成一句话：泛型类型在逻辑上看以看成是多个不同的类型，实际上都是相同的基本类型。**

```Java
List<String> stringArrayList = new ArrayList<String>();
List<Integer> integerArrayList = new ArrayList<Integer>();

Class classStringArrayList = stringArrayList.getClass();
Class classIntegerArrayList = integerArrayList.getClass();

if(classStringArrayList.equals(classIntegerArrayList)){
    Log.d("泛型测试","类型相同");
}
```

## 4. 泛型的使用
泛型有三种使用方式，分别为：泛型类、泛型接口、泛型方法。

### 4.1 泛型类
泛型类型用于类的定义中，被称为泛型类。通过泛型可以完成对一组类的操作对外开放相同的接口。最典型的就是各种容器类，如：`List`、`Set`、`Map`。

```Java
//此处T可以随便写为任意标识，常见的如T、E、K、V等形式的参数常用于表示泛型
//在实例化泛型类时，必须指定T的具体类型
public class Generic<T>{
    //key这个成员变量的类型为T,T的类型由外部指定  
    private T key;

    public Generic(T key) { //泛型构造方法形参key的类型也为T，T的类型由外部指定
        this.key = key;
    }

    public T getKey(){ //泛型方法getKey的返回值类型为T，T的类型由外部指定
        return key;
    }
}
```

定义的泛型类，就一定要传入泛型类型实参么？并不是这样，在使用泛型的时候如果传入泛型实参，则会根据传入的泛型实参做相应的限制，此时泛型才会起到本应起到的限制作用。如果不传入泛型类型实参的话，在泛型类中使用泛型的方法或成员变量定义的类型可以为任何的类型。

**注意：**
1. 泛型的类型参数只能是类类型，不能是简单类型。
2. 不能对确切的泛型类型使用`instanceof`操作。如下面的操作是非法的，编译时会出错。
    ```Java
    if(ex_num instanceof Generic<Number>){   }
    ```

### 4.2 泛型接口

```Java
//定义一个泛型接口
public interface Generator<T> {
    public T next();
}
```
当实现泛型接口的类，未传入泛型实参时：

```Java
/**
 * 未传入泛型实参时，与泛型类的定义相同，在声明类的时候，需将泛型的声明也一起加到类中
 * 即：class FruitGenerator<T> implements Generator<T>{
 * 如果不声明泛型，如：class FruitGenerator implements Generator<T>，编译器会报错："Unknown class"
 */
class FruitGenerator<T> implements Generator<T>{
    @Override
    public T next() {
        return null;
    }
}
```
当实现泛型接口的类，传入泛型实参时：
传入泛型实参时：
定义一个生产器实现这个接口,虽然我们只创建了一个泛型接口`Generator<T>`但是我们可以为T传入无数个实参，形成无数种类型的`Generator`接口。

在实现类实现泛型接口时，如已将泛型类型传入实参类型，则所有使用泛型的地方都要替换成传入的实参类型。即：`Generator<T>`，`public T next()`;中的的T都要替换成传入的`String`类型。

```Java

public class FruitGenerator implements Generator<String> {

    private String[] fruits = new String[]{"Apple", "Banana", "Pear"};

    @Override
    public String next() {
        Random rand = new Random();
        return fruits[rand.nextInt(3)];
    }
}
```
### 4.3 泛型通配符
我们知道I`ngeter`是`Number`的一个子类，我们验证过`Generic<Ingeter>`与`Generic<Number>`实际上是相同的一种基本类型。那么问题来了，在使用`Generic<Number>`作为形参的方法中，能否使用`Generic<Ingeter>`的实例传入呢？在逻辑上类似于`Generic<Number>`和`Generic<Ingeter>`是否可以看成具有父子关系的泛型类型呢?

```Java
ublic void showKeyValue1(Generic<Number> obj){
    Log.d("泛型测试","key value is " + obj.getKey());
}
```
```Java
Generic<Integer> gInteger = new Generic<Integer>(123);
Generic<Number> gNumber = new Generic<Number>(456);

showKeyValue(gNumber);

// showKeyValue这个方法编译器会为我们报错：Generic<java.lang.Integer>
// cannot be applied to Generic<java.lang.Number>
// showKeyValue(gInteger);
```
通过提示信息我们可以看到`Generic<Integer>`不能被看作为`Generic<Number>`的子类。由此可以看出:同一种泛型可以对应多个版本（因为参数类型是不确定的），不同版本的泛型类实例是不兼容的。

因此我们需要一个在逻辑上可以表示同时是`Generic<Integer>`和`Generic<Number>`父类的引用类型。由此类型通配符应运而生。

```Java
public void showKeyValue1(Generic<?> obj){
    Log.d("泛型测试","key value is " + obj.getKey());
}
```
类型通配符一般是使用`?`代替具体的类型实参，注意，**此处"?"是类型实参，而不是类型形参**。

再直白点的意思就是，此处的？和`Number`、`String`、`Integer`一样都是一种实际的类型，可以把？看成所有类型的父类。是一种真实的类型。

可以解决当具体类型不确定的时候，这个通配符就是`?`；当操作类型时，不需要使用类型的具体功能时，只使用`Object`类中的功能。那么可以用 `?`通配符来表未知类型。

### 4.4 泛型方法
我们见到的大多数泛型类中的成员方法也都使用了泛型，有的甚至泛型类中也包含着泛型方法，这样在初学者中非常容易将泛型方法理解错了。

泛型类，是在实例化类的时候指明泛型的具体类型；泛型方法，是在调用方法的时候指明泛型的具体类型 。
```Java
/**
 * 泛型方法的基本介绍
 * @param tClass 传入的泛型实参
 * @return T 返回值为T类型
 * 说明：
 *     1）public 与 返回值中间<T>非常重要，可以理解为声明此方法为泛型方法。
 *     2）只有声明了<T>的方法才是泛型方法，泛型类中的使用了泛型的成员方法并不是泛型方法。
 *     3）<T>表明该方法将使用泛型类型T，此时才可以在方法中使用泛型类型T。
 *     4）与泛型类的定义一样，此处T可以随便写为任意标识，常见的如T、E、K、V等形式的参数常用于表示泛型。
 */
public <T> T genericMethod(Class<T> tClass)throws InstantiationException ,
  IllegalAccessException{
        T instance = tClass.newInstance();
        return instance;
}
```
#### 4.4.1 泛型方法的基本用法
```Java
public class GenericTest {

   //这个类是个泛型类，在上面已经介绍过
   public class Generic<T>{     
        private T key;

        public Generic(T key) {
            this.key = key;
        }

        //我想说的其实是这个，虽然在方法中使用了泛型，但是这并不是一个泛型方法。
        //这只是类中一个普通的成员方法，只不过他的返回值是在声明泛型类已经声明过的泛型。
        //所以在这个方法中才可以继续使用 T 这个泛型。
        public T getKey(){
            return key;
        }

        /**
         * 这个方法显然是有问题的，在编译器会给我们提示这样的错误信息"cannot reslove symbol E"
         * 因为在类的声明中并未声明泛型E，所以在使用E做形参和返回值类型时，编译器会无法识别。
        public E setKey(E key){
             this.key = keu
        }
        */
    }

    /**
     * 这才是一个真正的泛型方法。
     * 首先在public与返回值之间的<T>必不可少，这表明这是一个泛型方法，并且声明了一个泛型T
     * 这个T可以出现在这个泛型方法的任意位置.
     * 泛型的数量也可以为任意多个
     *    如：public <T,K> K showKeyName(Generic<T> container){
     *        ...
     *        }
     */
    public <T> T showKeyName(Generic<T> container){
        System.out.println("container key :" + container.getKey());
        //当然这个例子举的不太合适，只是为了说明泛型方法的特性。
        T test = container.getKey();
        return test;
    }

    //这也不是一个泛型方法，这就是一个普通的方法，只是使用了Generic<Number>这个泛型类做形参而已。
    public void showKeyValue1(Generic<Number> obj){
        Log.d("泛型测试","key value is " + obj.getKey());
    }

    //这也不是一个泛型方法，这也是一个普通的方法，只不过使用了泛型通配符?
    //同时这也印证了泛型通配符章节所描述的，?是一种类型实参，可以看做为Number等所有类的父类
    public void showKeyValue2(Generic<?> obj){
        Log.d("泛型测试","key value is " + obj.getKey());
    }

     /**
     * 这个方法是有问题的，编译器会为我们提示错误信息："UnKnown class 'E' "
     * 虽然我们声明了<T>,也表明了这是一个可以处理泛型的类型的泛型方法。
     * 但是只声明了泛型类型T，并未声明泛型类型E，因此编译器并不知道该如何处理E这个类型。
    public <T> T showKeyName(Generic<E> container){
        ...
    }  
    */

    /**
     * 这个方法也是有问题的，编译器会为我们提示错误信息："UnKnown class 'T' "
     * 对于编译器来说T这个类型并未项目中声明过，因此编译也不知道该如何编译这个类。
     * 所以这也不是一个正确的泛型方法声明。
    public void showkey(T genericObj){

    }
    */
}
```

#### 4.4.2 类中的泛型方法
当然这并不是泛型方法的全部，泛型方法可以出现杂任何地方和任何场景中使用。但是有一种情况是非常特殊的，当泛型方法出现在泛型类中时:
```Java
public class GenericFruit {
    class Fruit{
        @Override
        public String toString() {
            return "fruit";
        }
    }

    class Apple extends Fruit{
        @Override
        public String toString() {
            return "apple";
        }
    }

    class Person{
        @Override
        public String toString() {
            return "Person";
        }
    }

    class GenerateTest<T>{
        public void show_1(T t){
            System.out.println(t.toString());
        }

        //在泛型类中声明了一个泛型方法，使用泛型E，这种泛型E可以为任意类型。可以类型与T相同，也可以不同。
        //由于泛型方法在声明的时候会声明泛型<E>，因此即使在泛型类中并未声明泛型，编译器也能够正确识别泛型方法中识别的泛型。
        public <E> void show_3(E t){
            System.out.println(t.toString());
        }

        //在泛型类中声明了一个泛型方法，使用泛型T，注意这个T是一种全新的类型，可以与泛型类中声明的T不是同一种类型。
        public <T> void show_2(T t){
            System.out.println(t.toString());
        }
    }

    public static void main(String[] args) {
        Apple apple = new Apple();
        Person person = new Person();

        GenerateTest<Fruit> generateTest = new GenerateTest<Fruit>();
        //apple是Fruit的子类，所以这里可以
        generateTest.show_1(apple);
        //编译器会报错，因为泛型类型实参指定的是Fruit，而传入的实参类是Person
        //generateTest.show_1(person);

        //使用这两个方法都可以成功
        generateTest.show_2(apple);
        generateTest.show_2(person);

        //使用这两个方法也都可以成功
        generateTest.show_3(apple);
        generateTest.show_3(person);
    }
}
```
#### 4.4.3 泛型方法与可变参数


#### 4.4.4 静态方法与泛型
静态方法有一种情况需要注意一下，那就是在类中的静态方法使用泛型：静态方法无法访问类上定义的泛型；如果静态方法操作的引用数据类型不确定的时候，必须要将泛型定义在方法上。

即：如果静态方法要使用泛型的话，必须将静态方法也定义成泛型方法 。
```Java
public class StaticGenerator<T> {
    ....
    ....
    /**
     * 如果在类中定义使用泛型的静态方法，需要添加额外的泛型声明（将这个方法定义成泛型方法）
     * 即使静态方法要使用泛型类中已经声明过的泛型也不可以。
     * 如：public static void show(T t){..},此时编译器会提示错误信息：
          "StaticGenerator cannot be refrenced from static context"
     */
    public static <T> void show(T t){

    }
}
```
#### 4.4.5 泛型与多态
下面来考虑一下泛型中多态问题。普通类型的多态是通过继承并重写父类的方法来实现的，泛型也不例外，下面是一个泛型多态示例。
```Java
package demo.blog.java.generic;

public class Father<T> {
    public void set(T t) {
        System.out.println("I am father, t=" + t);
    }
    public T get() {
        return null;
    }
}
```

```Java
public class Son extends Father<String> {

    @Override
    public void set(String t) {
        super.set(t);
        System.out.println("I am son.");
    }

    @Override
    public String get() {
        return super.get();
    }

    public static void main(String[] args) {
        Father<String> father = new Son();
        father.set("hello world");
    }
}
```
上面定义了一个泛型父类和一个实际参数为`String`类型的子类，并“重写”了`set(T)`和`get()`方法。`Son`类中的`@Override`注解也清楚地显示这是一个重写方法，最终执行的结果如下，与想象中的结果完全一致。
>I am father, t=hello world
>I am son.

真的这么简单么？虽然表面上（源代码层面）来看，泛型多态与普通类的多态并无二样，但是其内部的实时原理却大相径庭。
泛型类`Father`在编译后会擦除泛型信息，所有的泛型参数都会用`Object`类替代。实际上，`Father`编译后的字节码与下面的代码完全一致。
```Java
public class Father {
    public void set(Object t) {
        System.out.println("I am father, t=" + t);
    }
    public Object get() {
        return null;
    }
}
```
Son类的与最终会变为：
```Java
public class Son extends Father {

    @Override
    public void set(String t) {
        super.set(t);
        System.out.println("I am son.");
    }

    @Override
    public String get() {
        return super.get();
    }

    public static void main(String[] args) {
        Father father = new Son();
        father.set("hello world");
    }
}
```
`Father`和`Son`类的`set()`方法的参数类型不一样，所以，这并不是方法重写，而是方法重载！但是，如果是重载，那么`Son`类就应该会继承`Father`类的`set(Object)`方法，也就是Son会同时包含`set(String)`和`set(Object)`，下面来测试一下。
```Java
Son son = new Son();
son.set("test");
son.set(new Object()); // 编译错误
```
当`set`一个`Object`对象时，编译无法通过。这就很奇怪了，感觉跟之前学到的知识是相悖的。我们原本想通过重写方法来实现多态，但由于泛型的类型擦除，却最终变成了重载，所以类型擦除与多态有了矛盾。那么Java是怎么解决这个问题的呢？还是从字节码中找答案吧。`Son`类最终的编译结果如下：
```
{
  public demo.blog.java.generic.Son();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #8                  // Method demo/blog/java/generic/Father."<init>":()V
         4: return
      LineNumberTable:
        line 13: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Ldemo/blog/java/generic/Son;

  public void set(java.lang.String);         // 我们重写的方法
    descriptor: (Ljava/lang/String;)V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: invokespecial #16                 // Method demo/blog/java/generic/Father.set:(Ljava/lang/Object;)V
         5: getstatic     #19                 // Field java/lang/System.out:Ljava/io/PrintStream;
         8: ldc           #25                 // String I am son.
        10: invokevirtual #27                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        13: return
      LineNumberTable:
        line 17: 0
        line 18: 5
        line 19: 13
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      14     0  this   Ldemo/blog/java/generic/Son;
            0      14     1     t   Ljava/lang/String;

  public java.lang.String get();              // 我们重写的方法
    descriptor: ()Ljava/lang/String;
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #36                 // Method demo/blog/java/generic/Father.get:()Ljava/lang/Object;
         4: checkcast     #39                 // class java/lang/String
         7: areturn
      LineNumberTable:
        line 23: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       8     0  this   Ldemo/blog/java/generic/Son;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: new           #1                  // class demo/blog/java/generic/Son
         3: dup
         4: invokespecial #43                 // Method "<init>":()V
         7: astore_1
         8: aload_1
         9: ldc           #44                 // String hello world
        11: invokevirtual #16                 // Method demo/blog/java/generic/Father.set:(Ljava/lang/Object;)V
        14: return
      LineNumberTable:
        line 27: 0
        line 28: 8
        line 29: 14
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      15     0  args   [Ljava/lang/String;
            8       7     1 father   Ldemo/blog/java/generic/Father;
      LocalVariableTypeTable:
        Start  Length  Slot  Name   Signature
            8       7     1 father   Ldemo/blog/java/generic/Father<Ljava/lang/String;>;

  public java.lang.Object get();              // 编译器生成的方法
    descriptor: ()Ljava/lang/Object;
    flags: ACC_PUBLIC, ACC_BRIDGE, ACC_SYNTHETIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokevirtual #52                 // Method get:()Ljava/lang/String;
         4: areturn
      LineNumberTable:
        line 1: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature

  public void set(java.lang.Object);          // 编译器生成的方法
    descriptor: (Ljava/lang/Object;)V
    flags: ACC_PUBLIC, ACC_BRIDGE, ACC_SYNTHETIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: checkcast     #39                 // class java/lang/String
         5: invokevirtual #54                 // Method set:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 1: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
}

```

编译结果有点长，为了方便，省去了无关的字节码。先注意到第77行和第90行，这里面多了一个`Object get()`方法和`set(Object)`方法，这两个方法在`Son`类源代码里面并不存在，这是编译器为了解决泛型的多态问题而自动生成的方法，称为“桥方法”。这两个方法的签名与Father类中的两个方法的签名完全一致，这才是真正的方法重写。也就是说，子类真正重写的我们看不到的桥方法，`@Override`注解只是假象，让人误以为他们真的是重写方法。

再看看`set(Object)`桥方法的实现细节，在第97行处先将`Object`对象强制转换为`String`对象，然后调用`Son`中的`set(String)`方法。饶了一个圈，最终才回到我们“重写”的方法。main方法中原本调用父类的`set(Object)`方法，由于子类通过桥方法重写了这个方法，所以最终的调用顺序是：`set(Object)` -> `set(String)`。

这是脑海中必然会闪现出这么一个问题，既然调用的是`set(Object)`方法，那么可以在源代码中`set`一个`Object`对象么？下面来测试一下：
```Java
Father<String> father = new Son();
father.set("hello world");
father.set(new Object()); // 编译错误
```
上面会出现编译错误，如果是在运行时环境中，这样是可以运行的，不过会出现`ClassCastException`异常。

与`set(Object)`桥方法的意义不同，`Object get()`并不仅仅解决泛型与重写的冲突，而更具有一般性。看看下面的代码，这是一个普通类的继承，

```Java
public class GeneralFather {

    public Object get() {
        return null;
    }
}
```

```Java
public class GeneralSon extends GeneralFather {

    @Override
    public String get() {
        return "";
    }
}
```

子类的返回类型是父类的返回类型的子类，这是允许的，这种特性叫做`Java`返回值的协变性。而协变性的实现方法就是上面所述的桥方法。

这里还会有疑惑`，set`方法可以通过参数类型来确定调用的方法。但是，参数一样而返回值不一样是不能重载的。如果我们在源代码中通过编写`String get()`和`Object get()`方法是无法通过编译的。虽然，编译器无法通过编译，但是`JVM`是可以编写这两种方法的，它调用方法时，将返回值也作为方法签名的一部分。有种只许州官放火，不许百姓点灯的感觉。可以看到，`JVM`做了不少我们认为不合法的事情，所以如果不深入研究底层原理，有些问题根本解释不了。

#### 4.4.6 泛型方法总结
泛型方法能使方法独立于类而产生变化，以下是一个基本的指导原则：

无论何时，如果你能做到，你就该尽量使用泛型方法。也就是说，如果使用泛型方法将整个类泛型化，那么就应该使用泛型方法。另外对于一个`static`的方法而已，无法访问泛型类型的参数。所以如果`static`方法要使用泛型能力，就必须使其成为泛型方法。

### 4.5 泛型上下边界
在使用泛型的时候，我们还可以为传入的泛型类型实参进行上下边界的限制，如：类型实参只准传入某种类型的父类或某种类型的子类。

为泛型添加上边界，即传入的类型实参必须是指定类型的子类型。
```Java
public void showKeyValue1(Generic<? extends Number> obj){
    Log.d("泛型测试","key value is " + obj.getKey());
}
```

```Java
Generic<String> generic1 = new Generic<String>("11111");
Generic<Integer> generic2 = new Generic<Integer>(2222);
Generic<Float> generic3 = new Generic<Float>(2.4f);
Generic<Double> generic4 = new Generic<Double>(2.56);

//这一行代码编译器会提示错误，因为String类型并不是Number类型的子类
//showKeyValue1(generic1);

showKeyValue1(generic2);
showKeyValue1(generic3);
showKeyValue1(generic4);
```
如果我们把泛型类的定义也改一下:
```Java
public class Generic<T extends Number>{
    private T key;

    public Generic(T key) {
        this.key = key;
    }

    public T getKey(){
        return key;
    }
}
```

```Java
//这一行代码也会报错，因为String不是Number的子类
Generic<String> generic1 = new Generic<String>("11111");
```
再来一个泛型方法的例子：
```Java
//在泛型方法中添加上下边界限制的时候，必须在权限声明与返回值之间的<T>上添加上下边界，即在泛型声明的时候添加
//public <T> T showKeyName(Generic<T extends Number> container)，编译器会报错："Unexpected bound"
public <T extends Number> T showKeyName(Generic<T> container){
    System.out.println("container key :" + container.getKey());
    T test = container.getKey();
    return test;
}
```
通过上面的两个例子可以看出：泛型的上下边界添加，必须与泛型的声明在一起 。
## 5. 泛型的实现原理
### 5.1 Java中的泛型是伪泛型
泛型思想最早在`C++`语言的模板`（Templates）`中产生，`Java`后来也借用了这种思想。虽然思想一致，但是他们存在着本质性的不同。`C++`中的模板是真正意义上的泛型，在编译时就将不同模板类型参数编译成对应不同的目标代码，`ClassName`和`ClassName`是两种不同的类型，这种泛型被称为真正泛型。这种泛型实现方式，会导致类型膨胀，因为要为不同具体参数生成不同的类。

`Java`中`ClassName`和`ClassName`虽然在源代码中属于不同的类，但是编译后的字节码中，他们都被替换成原始类型`（ClassName）`，而两者的原始类型的一样的，所以在运行时环境中，`ClassName`和`ClassName`就是同一个类。`Java`中的泛型是一种特殊的语法糖，通过类型擦除实现（后面介绍），这种泛型称为伪泛型。由于`Java`中有这么一个障眼法，如果没有进行深入研究，就会在产生莫名其妙的问题。值得一提的是，不少大牛对Java的泛型的实现方式很不满意。
### 5.2 类型擦除
`Java`中的泛型是通过类型擦除来实现的。所谓类型擦除，是指通过类型参数合并，将泛型类型实例关联到同一份字节码上。编译器只为泛型类型生成一份字节码，并将其实例关联到这份字节码上。类型擦除的关键在于从泛型类型中清除类型参数的相关信息，并且再必要的时候添加类型检查和类型转换的方法。

下面通过两个例子来证明在编译时确实发生了类型擦除。

分别创建实际类型为`String`和`Integer`的`ArrayList`对象，通过`getClass()`方法获取两个实例的类，最后判断这个实例的类是相等的，证明两个实例共享同一个类。
```Java
// 声明一个具体类型为String的ArrayList
ArrayList<String> arrayList1 = new ArrayList<String>();  
arrayList1.add("abc");  

// 声明一个具体类型为Integer的ArrayList
ArrayList<Integer> arrayList2 = new ArrayList<Integer>();  
arrayList2.add(123);  

System.out.println(arrayList1.getClass() == arrayList2.getClass());  // 结果为true
```
创建一个只能存储`Integer`的`ArrayList`对象，在add一个整型数值后，利用反射调用`add(Object o)``add`一个`asd`字符串，此时运行代码不会报错，运行结果会打印出1和`as`d两个值。这时再里利用反射调用`add(Integer o)`方法，运行会抛出`codeNoSuchMethodException`异常。这充分证明了在编译后，擦除了Integer这个泛型信息，只保留了原始类型。
```Java
ArrayList<Integer> arrayList3 = new ArrayList<Integer>();
arrayList3.add(1);
arrayList3.getClass().getMethod("add", Object.class).invoke(arrayList3, "asd");
for (int i = 0; i < arrayList3.size(); i++) {
    System.out.println(arrayList3.get(i)); // 输出1，asd
}

arrayList3.getClass().getMethod("add", Integer.class).invoke(arrayList3, 2); // NoSuchMethodException：java.util.ArrayList.add(java.lang.Integer)
```
### 5.3 自动类型转换
上一节上说到了类型擦除，`Java`编译器会擦除掉泛型信息。那么调用`ArrayList`的`get()`最终返回的必然会是一个`Object`对象，但是我们在源代码并没有写过`Object`转成`Integer`的代码，为什么就能“直接”将取出来的对象赋予一个`Integer`类型的变量呢
```Java
import java.util.List;
import java.util.ArrayList;

/**
 * 泛型中的类型转换测试。
 */
public class Test {

    public static void main(String[] args) {
        List<Integer> a = new ArrayList<Integer>();
        a.add(1);
        Integer ai = a.get(0);
    }
}
```
实际上，`Java`的泛型除了类型擦除之外，还会自动生成`checkcast`指令进行强制类型转换。上面的代码中的`main`方法编译后所对应的字节码如下。
```
public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: new           #2                  // class java/util/ArrayList
         3: dup
         4: invokespecial #3                  // Method java/util/ArrayList."<init>":()V
         7: astore_1
         8: aload_1
         9: iconst_1
        10: invokestatic  #4                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        13: invokeinterface #5,  2            // InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z
        18: pop
        19: aload_1
        20: iconst_0
        21: invokeinterface #6,  2            // InterfaceMethod java/util/List.get:(I)Ljava/lang/Object;
        26: checkcast     #7                  // class java/lang/Integer
        29: astore_2
        30: return
      LineNumberTable:
        line 7: 0
        line 8: 8
        line 9: 19
        line 10: 30
}

```

看到第18行代码就是将`Object`类型的对象强制转换为`Integer`的指令。我们完全可以将上面的代码转换为下面的代码，它所实现的效果跟上面的泛型是一模一样的。既然泛型也需要进行强制转换，所以泛型并不会提供运行时效率，不过可以大大降低编程时的出错概率。

```Java
public static void main(String[] args) {
    List a = new ArrayList();
    a.add(1);
    Integer ai = (Integer)a.get(0);
}
```

## 6. 关于泛型数组要提一下

### 6.1 泛型与数组
看到了很多文章中都会提起泛型数组，经过查看`sun`的说明文档，在`java`中是”不能创建一个确切的泛型类型的数组”的。

也就是说下面的这个例子是不可以的：
```Java
List<String>[] ls = new ArrayList<String>[10];  
```
而使用通配符创建泛型数组是可以的，如下面这个例子：
```Java
List<?>[] ls = new ArrayList<?>[10];  
```
这样也是可以的：
```Java
ist<String>[] ls = new ArrayList[10];
```
下面使用一个例子来说明这个问题
```Java
List<String>[] lsa = new List<String>[10]; // Not really allowed.    
Object o = lsa;    
Object[] oa = (Object[]) o;    
List<Integer> li = new ArrayList<Integer>();    
li.add(new Integer(3));    
oa[1] = li; // Unsound, but passes run time store check    
String s = lsa[1].get(0); // Run-time error: ClassCastException.
```
这种情况下，由于JVM泛型的擦除机制，在运行时JVM是不知道泛型信息的，所以可以给`oa[1]`赋上一个`ArrayList`而不会出现异常，但是在取出数据的时候却要做一次类型转换，所以就会出现`ClassCastException`，如果可以进行泛型数组的声明，上面说的这种情况在编译期将不会出现任何的警告和错误，只有在运行时才会出错。

而对泛型数组的声明进行限制，对于这样的情况，可以在编译期提示代码有类型安全问题，比没有任何提示要强很多。

下面采用通配符的方式是被允许的:数组的类型不可以是类型变量，除非是采用通配符的方式，因为对于通配符的方式，最后取出数据是要做显式的类型转换的。

```Java
List<?>[] lsa = new List<?>[10]; // OK, array of unbounded wildcard type.    
Object o = lsa;    
Object[] oa = (Object[]) o;    
List<Integer> li = new ArrayList<Integer>();    
li.add(new Integer(3));    
oa[1] = li; // Correct.    
Integer i = (Integer) lsa[1].get(0); // OK
```

### 6.2 泛型与静态方法和静态类
泛型类中的静态变量和静态方法和不可以使用泛型类所声明的泛型类型参数，下面的操作是不合法的。
```Java
public class Test<T> {  
    public static T one;   //编译错误  

    public static  T show(T one){ //编译错误  
        return null;  
    }  
}  
```
这是由于泛型的具体参数要在实例化是才能确定，而静态变量和静态方法无需实例化就可以调用。当对象都还没有创建时，就调用与泛型变量相关的方法，当然是错误的。不过，对于泛型方法，静态泛型方法是可以的，因为具体的泛型类型无需实例化就可以确定。

### 6.3 方法命名冲突
下面来看看由于使用泛型而引起的方法命名冲突问题。
```Java
public class Father<T> {

    // 编译不通过，Name clash: The method equals(T) of type Father<T> has the same erasure as equals(Object) of type Object but does not override it
    public boolean equals(T obj) {
        // TODO Auto-generated method stub
        return super.equals(obj);
    }
}
```
我们的本意是想写一个与指定泛型类型对象判断相等性的方法，这个方法与`Object`的`equals(Object)`方法的含义本质上是不同的。`Object`的`equals(Object)`是判断相同类型的对象的相等，而我们定义的方法却不是这个含义。

以`Father`为例，看上去重载了`Object`的`equals(Object)`方法，因为我们定义的equals方法的参数是String类型的，也就是equals(String)。但是，由于Father在编译后会擦除泛型信息，所以equals(T)就被编译成了equals(Object)，这与Object的equals(Object)方法签名一模一样，这不就是覆盖么！我们并不想覆盖父类的方法，却在事实上覆盖了。如果上面的代码可以编译并运行，对于开发者来说就是一个大坑。唯一的办法就是改另外一个方法名。

**总结**
本文中的例子主要是为了阐述泛型中的一些思想而简单举出的，并不一定有着实际的可用性。另外，一提到泛型，相信大家用到最多的就是在集合中，其实，在实际的编程过程中，自己可以使用泛型去简化开发，且能很好的保证代码质量。
