>`面向对象`编程是对`数据`进行抽象，而`函数式`编程是对`行为`进行抽象
===============================================
##Lambda表达式
------------------------------------
Lambda表达式的不同形式
1. Runnable noArguments = () ->System.out.println("Hi");
2. ActionListener oneArgument = event ->System.out.println("button clicked");
3. Runnable multiStatement = () -> {
  System.out.println("Hello");
  System.out.println("World");};
4. BinaryOperator<Long> add = (x,y) -> x + y
5. BinaryOperator<Long> addExplicit = (Long x, Long y) -> x + y;

> Lambda表达式的类型依赖于上下文环境， 是由编译器推断出来的。
> Lambda表达式中引用的局部变量必须是final或既成事实上的fianl变量

>函数接口是只有一个抽象方法的接口，用作Lambada表达式的类型


`Stream.of()可以包含多个Stream`


`Reduce操作可以实现都一组值中生成一个值`
```
int count = Stream.of(1,2,3).reduce(0,(acc,element)-> acc + element)
```

Lambda使用Stream（内部迭代）控制权更多的交给了集合类
基本类型定制的Lambda或Stream可以提高性能
在一个值可能为空的建模情况下，使用Optional对象能替代使用null值


new object:
```
String[]::new
Artist::new
```
