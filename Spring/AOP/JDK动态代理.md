
## JDK 动态代理

相比于静态代理，动态代理避免了开发人员编写各个繁锁的静态代理类，只需简单地指定一组接口及目标类对象就能动态的获得代理对象。

#### 代理模式

使用代理模式必须要让代理类和目标类实现相同的接口，客户端通过代理类来调用目标方法，代理类会将所有的方法调用分派到目标对象上反射执行，还可以在分派过程中添加"前置通知"和后置处理（如在调用目标方法前校验权限，在调用完目标方法后打印日志等）等功能。

![动态代理][1]

#### demo.
你可以在通过[github][2]看到源码。

```Java
public interface Animal {
    String sound();
}

public class Dog implements Animal {
    @Override
    public String sound() {
        return "Wang, Wang, Wang";
    }
}
```

通过实现InvocationHandler接口来自定义自己的InvocationHandler

```Java
public class DogProxyHandler implements InvocationHandler {

    private Object proxyied; //代理对象

    public DogProxyHandler(Object proxyied){
        this.proxyied = proxyied;  //初始化代理对象
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before invoked...");
        System.out.println(method.getName() + method.getModifiers());

        Object object = method.invoke(proxyied, args);

        System.out.println("after invoked...");

        return object;
    }
}
```
client调用过程。

```Java
public class Test {
    public static void main(String[] args) {
        Dog dog = new Dog();

        Animal animal = (Animal) Proxy.newProxyInstance(
                        Animal.class.getClassLoader(),
                        new Class[]{Animal.class},
                        new DogProxyHandler(dog));

        System.out.println(animal.sound());

    }
}
```

console 打印输出：
```Java
before invoked...
Wang, Wang, Wang
after invoked...
```

其实这里基本上就是AOP的一个简单实现了，在目标对象的方法执行之前和执行之后进行了增强。Spring的AOP实现其实也是用了Proxy和InvocationHandler这两个类来操作的。

---

###源码分析
**1.newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h)**
```Java
@CallerSensitive
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
    throws IllegalArgumentException
{
    Objects.requireNonNull(h);

    final Class<?>[] intfs = interfaces.clone();
    final SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }

    /*
     * Look up or generate the designated proxy class.
     */
     //核心在这里，这里生成了最关键的代理类。
    Class<?> cl = getProxyClass0(loader, intfs);

    /*
     * Invoke its constructor with the designated invocation handler.
     */
    try {
        if (sm != null) {
            checkNewProxyPermission(Reflection.getCallerClass(), cl);
        }

        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        if (!Modifier.isPublic(cl.getModifiers())) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
        }
        return cons.newInstance(new Object[]{h});
    } catch (IllegalAccessException|InstantiationException e) {
        throw new InternalError(e.toString(), e);
    } catch (InvocationTargetException e) {
        Throwable t = e.getCause();
        if (t instanceof RuntimeException) {
            throw (RuntimeException) t;
        } else {
            throw new InternalError(t.toString(), t);
        }
    } catch (NoSuchMethodException e) {
        throw new InternalError(e.toString(), e);
    }
}

```

在这里，我们要关心的是`ProxyClassFactory`具体做了什么。

**2.ProxyClassFactory.apply(ClassLoader loader, Class<?>[] interfaces)**
```Java

    /**
     * A factory function that generates, defines and returns the proxy class given
     * the ClassLoader and array of interfaces.
     */
    private static final class ProxyClassFactory
        implements BiFunction<ClassLoader, Class<?>[], Class<?>>
    {
        // prefix for all proxy class names
        private static final String proxyClassNamePrefix = "$Proxy";

        // next number to use for generation of unique proxy class names
        private static final AtomicLong nextUniqueNumber = new AtomicLong();

        @Override
        public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

            Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
            for (Class<?> intf : interfaces) {
                /*
                 * Verify that the class loader resolves the name of this
                 * interface to the same Class object.
                 */
                Class<?> interfaceClass = null;
                try {
                    interfaceClass = Class.forName(intf.getName(), false, loader);
                } catch (ClassNotFoundException e) {
                }
                if (interfaceClass != intf) {
                    throw new IllegalArgumentException(
                        intf + " is not visible from class loader");
                }
                /*
                 * Verify that the Class object actually represents an
                 * interface.
                 */
                if (!interfaceClass.isInterface()) {
                    throw new IllegalArgumentException(
                        interfaceClass.getName() + " is not an interface");
                }
                /*
                 * Verify that this interface is not a duplicate.
                 */
                if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                    throw new IllegalArgumentException(
                        "repeated interface: " + interfaceClass.getName());
                }
            }

            String proxyPkg = null;     // package to define proxy class in
            int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

            /*
             * Record the package of a non-public proxy interface so that the
             * proxy class will be defined in the same package.  Verify that
             * all non-public proxy interfaces are in the same package.
             */
             /*验证你传入的接口中是否有非public接口，只要有一个接口是非public的，那么这些接口都必须在同一包中
              这里的接口修饰符直接影响到System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true")所生成
              的代理类的路径，往下看！！
            */
            for (Class<?> intf : interfaces) {
                int flags = intf.getModifiers();
                if (!Modifier.isPublic(flags)) {
                    accessFlags = Modifier.FINAL;
                    String name = intf.getName();
                    int n = name.lastIndexOf('.');
                    String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                    if (proxyPkg == null) {
                        proxyPkg = pkg;
                    } else if (!pkg.equals(proxyPkg)) {
                        throw new IllegalArgumentException(
                            "non-public interfaces from different packages");
                    }
                }
            }

            if (proxyPkg == null) {
                // if no non-public proxy interfaces, use com.sun.proxy package
                proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
            }

            /*
             * Choose a name for the proxy class to generate.
             */
            long num = nextUniqueNumber.getAndIncrement();
            String proxyName = proxyPkg + proxyClassNamePrefix + num;

            /*
             * Generate the specified proxy class.
             */
             //这里是关键的生成字节码文件的地方。
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
            try {
                return defineClass0(loader, proxyName,
                                    proxyClassFile, 0, proxyClassFile.length);
            } catch (ClassFormatError e) {
                /*
                 * A ClassFormatError here means that (barring bugs in the
                 * proxy class generation code) there was some other
                 * invalid aspect of the arguments supplied to the proxy
                 * class creation (such as virtual machine limitations
                 * exceeded).
                 */
                throw new IllegalArgumentException(e.toString());
            }
        }
    }

```
我们可以看到，这个方法主要是做了一些验证性工作，做了一些准备工作，生成字节码的工作，主要是交给了`ProxyGenerator`的`generateProxyClass`来做。
**3.ProxyGenerator.generateProxyClass(proxyName, interfaces, accessFlags);**

```Java
    private static final boolean saveGeneratedFiles =
      ((Boolean)AccessController.doPrivileged(new GetBooleanAction("sun.misc.ProxyGenerator.saveGeneratedFiles"))).booleanValue();

    public static byte[] generateProxyClass(final String var0, Class<?>[] var1, int var2) {
        ProxyGenerator var3 = new ProxyGenerator(var0, var1, var2);
        //真正的生成字节码byte.
        final byte[] var4 = var3.generateClassFile();
        //这里对应是否保存文件。sun.misc.ProxyGenerator.saveGeneratedFiles == true
        if (saveGeneratedFiles) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    try {
                        int var1 = var0.lastIndexOf(46);Path var2;
                        if (var1 > 0) {
                            Path var3 = Paths.get(var0.substring(0, var1).replace('.', File.separatorChar));
                            Files.createDirectories(var3);
                            var2 = var3.resolve(var0.substring(var1 + 1, var0.length()) + ".class");
                        } else {var2 = Paths.get(var0 + ".class");}
                        Files.write(var2, var4, new OpenOption[0]);
                        return null;
                    } catch (IOException var4x) {throw new InternalError("I/O exception saving generated file: " + var4x);}
                }
            });
        }
        return var4;
    }
```
层层调用后，最终`generateClassFile`才是真正生成代理类字节码文件的方法，注意开头的三个`addProxyMethod`方法是只将`Object`的`hashcode`,`equals`,`toString`方法添加到代理方法容器中，代理类除此之外并没有重写其他`Object`的方法，所以除这三个方法外，代理类调用其他方法的行为与`Object`调用这些方法的行为一样不通过Invoke.

**4.generateClassFile--根源**
```Java
    private byte[] generateClassFile() {
            /*addProxyMethod系列方法就是将接口的方法和Object的hashCode,equals,toString方法添加到代理方法容器(proxyMethods),
       其中方法签名作为key,proxyMethod作为value*/
            /*hashCodeMethod方法位于静态代码块中通过Object对象获得，hashCodeMethod=Object.class.getMethod("hashCode",new Class[0]),
       相当于从Object中继承过来了这三个方法equalsMethod,toStringMethod*/    
        this.addProxyMethod(hashCodeMethod, Object.class);
        this.addProxyMethod(equalsMethod, Object.class);
        this.addProxyMethod(toStringMethod, Object.class);


         //获得所有接口中的所有方法，并将方法添加到代理方法中
        Class[] var1 = this.interfaces;
        int var2 = var1.length;
        int var3;
        Class var4;
        for(var3 = 0; var3 < var2; ++var3) {
            var4 = var1[var3];
            Method[] var5 = var4.getMethods();
            int var6 = var5.length;

            for(int var7 = 0; var7 < var6; ++var7) {
                Method var8 = var5[var7];
                this.addProxyMethod(var8, var4);
            }
        }

        //验证具有相同方法签名的的方法的返回值类型是否一致，因为不可能有两个方法名相同,参数相同，而返回值却不同的方法
        Iterator var11 = this.proxyMethods.values().iterator();
        List var12;
        while(var11.hasNext()) {
            var12 = (List)var11.next();
            checkReturnTypes(var12);
        }

        //接下来就是写代理类文件的步骤了
        Iterator var15;
        try {
             //生成代理类的构造函数
            this.methods.add(this.generateConstructor());
            var11 = this.proxyMethods.values().iterator();

            while(var11.hasNext()) {
                var12 = (List)var11.next();
                var15 = var12.iterator();

                while(var15.hasNext()) {
                    ProxyGenerator.ProxyMethod var16 = (ProxyGenerator.ProxyMethod)var15.next();
                    /*将代理字段声明为Method，10为ACC_PRIVATE和ACC_STATAIC的与运算，表示该字段的修饰符为private static
                     所以代理类的字段都是private static Method */
                    this.fields.add(new ProxyGenerator.FieldInfo(var16.methodFieldName, "Ljava/lang/reflect/Method;", 10));
                    //生成代理类的代理方法
                    this.methods.add(var16.generateMethod());
                }
            }

            //为代理类生成静态代码块，对一些字段进行初始化
            this.methods.add(this.generateStaticInitializer());
        } catch (IOException var10) {
            throw new InternalError("unexpected I/O Exception", var10);
        }

        if (this.methods.size() > 65535) {
            throw new IllegalArgumentException("method limit exceeded");
        } else if (this.fields.size() > 65535) {
            throw new IllegalArgumentException("field limit exceeded");
        } else {

          //代理类文件过程。
            this.cp.getClass(dotToSlash(this.className));
            this.cp.getClass("java/lang/reflect/Proxy");
            var1 = this.interfaces;
            var2 = var1.length;

            for(var3 = 0; var3 < var2; ++var3) {
                var4 = var1[var3];
                this.cp.getClass(dotToSlash(var4.getName()));
            }

            this.cp.setReadOnly();
            ByteArrayOutputStream var13 = new ByteArrayOutputStream();
            DataOutputStream var14 = new DataOutputStream(var13);

            try {
                var14.writeInt(-889275714);
                var14.writeShort(0);
                var14.writeShort(49);
                this.cp.write(var14);
                var14.writeShort(this.accessFlags);
                var14.writeShort(this.cp.getClass(dotToSlash(this.className)));
                var14.writeShort(this.cp.getClass("java/lang/reflect/Proxy"));
                var14.writeShort(this.interfaces.length);
                Class[] var17 = this.interfaces;
                int var18 = var17.length;

                for(int var19 = 0; var19 < var18; ++var19) {
                    Class var22 = var17[var19];
                    var14.writeShort(this.cp.getClass(dotToSlash(var22.getName())));
                }

                var14.writeShort(this.fields.size());
                var15 = this.fields.iterator();

                while(var15.hasNext()) {
                    ProxyGenerator.FieldInfo var20 = (ProxyGenerator.FieldInfo)var15.next();
                    var20.write(var14);
                }

                var14.writeShort(this.methods.size());
                var15 = this.methods.iterator();

                while(var15.hasNext()) {
                    ProxyGenerator.MethodInfo var21 = (ProxyGenerator.MethodInfo)var15.next();
                    var21.write(var14);
                }

                var14.writeShort(0);
                return var13.toByteArray();
            } catch (IOException var9) {
                throw new InternalError("unexpected I/O Exception", var9);
            }
        }
    }
```

在动态代理中InvocationHandler是核心，每个代理实例都具有一个关联的调用处理程序`InvocationHandler`。对代理实例调用方法时，将对方法调用进行编码并将其指派到它的调用处理程序`InvocationHandler`的 `invoke` 方法。所以对代理方法的调用都是通`InvocationHadler`的`invoke`来实现中，而`invoke`方法根据传入的代理对象，方法和参数来决定调用代理的哪个方法`invoke`方法签名：`invoke（Object Proxy，Method method，Object[] args）`

**5.生成的代理类**
```Java
public final class $Proxy0 extends Proxy implements Animal {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    //代理类的构造函数，其参数正是是InvocationHandler实例，Proxy.newInstance方法就是通过通过这个构造函数来创建代理实例的
    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return ((Boolean)super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String sound() throws  {
        try {
            return (String)super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return ((Integer)super.h.invoke(this, m0, (Object[])null)).intValue();
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("com.proxy.com.Animal").getMethod("sound");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}

```




[1]:https://images2015.cnblogs.com/blog/776259/201606/776259-20160618235740620-555209026.png
[2]:https://github.com/twentyworld/learn/tree/master/SpringBootAop/src/main/java/com/proxy/com
