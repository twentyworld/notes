# Jackson
Jackson是用于处理的JSON的类库。

---
### 一 开始之前
Java处理JSON数据有三个比较流行的类库`FastJSON`、`Gson`和`Jackson`。

- Jackson

>Jackson是由其社区进行维护，简单易用并且性能也相对高些。但是对于复杂的bean转换Json，转换的格式比较标准的Json格式。PS:Jackson为Spring MVC内置Json解析工具

- Gson
> Gson是由谷歌公司研发的产品，目前是最全的Json解析工具。完全可以将复杂的类型的Json解析成Bean或者Bean到Json的转换

- FastJson
> Fastjson是一个Java语言编写的高性能的JSON处理器,由阿里巴巴公司开发。FastJson采用独创的算法，将parse的速度提升到极致，超过所有json库。但是在对一些复杂类型的Bean转换Json上会出现一些问题，需要特殊处理。

---

首先我们定义两个类。
```Java
public class Person {
    private String name;
    private int age;
    private List<House> houses;

    public Person(String name, int age, List<House> houses) {
        this.name = name;
        this.age = age;
        this.houses = houses;
    }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
    public List<House> getHouses() { return houses; }
    public void setHouses(List<House> houses) { this.houses = houses; }
}

```

```Java
public class House {
    public int price;
    private String address;

    public House() {}

    public House(int price, String address) {
        this.price = price;
        this.address = address;
    }
}

```


## 二 注解

#### 1. @JsonAutoDetect

作用在类上,来开启/禁止自动检测。

注解的方法：**fieldVisibility** 字段的可见级别。

##### jackson默认的字段属性发现规则
- 所有被`public`修饰的字段
- 所有被`public`修饰的`getter`
- 所有被`public`修饰的`setter`

如果我们只有一个`private`的`name`属性，并且没有提供对应的`get,set`方法，如果按照默认的属性发现规则我们将无法序列化和反序列化`name`字段(如果没有`get,set`方法，只有被`public`修饰的属性才会被发现)。

示例：
我们修改一下`House`类：
```Java
public class House {
    public int price;
    private String address;

    public House() {}

    public House(int price, String address) {
        this.price = price;
        this.address = address;
    }
}

```
测试：
```Java
public static void main(String[] args) throws JsonProcessingException {
    ObjectMapper mapper = getObjectMapper();
    String json = mapper.writeValueAsString(getPerson());
    System.out.println(json);
}

public static Person getPerson() {
    House beijing = new House(300,"beijing");
    House shanghai = new House(400, "shanghai");

    return new Person("zed",24, Arrays.asList(beijing,shanghai));
}

public static ObjectMapper getObjectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    mapper.configure(FAIL_ON_EMPTY_BEANS,false);
    return mapper;
}
```
OutPut:
```Java
// 只有price， 没有Address。
{"name":"zed","age":24,"houses":[{"price":300},{"price":400}]}

```
可以通过修改`@JsonAutoDetect`的`fieldVisibility`来调整自动发现级别，为了使`name`被自动发现，我们需要将级别调整为`ANY`。

给`House`类添加注解`@JsonAutoDetect(fieldVisibility = JsonAutoDetect.Visibility.ANY)`：
OutPut:
```Java
// 有price， 有Address。
{"name":"zed","age":24,"houses":[{"price":300,"address":"beijing"},{"price":400,"address":"shanghai"}]}
```
同理，除了`fieldVisibility`可以设置外，还可以设置`getterVisibility`、`setterVisibility`、`isGetterVisibility`、`creatorVisibility`。

你可以配置`MapperFeature`来启动/禁止一些特别类型(`getters`,`setters``,fields`,`creators`)的自动检测。

例如：
```Java
public static ObjectMapper getObjectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    mapper.configure(FAIL_ON_EMPTY_BEANS,false);
    mapper.configure(MapperFeature.AUTO_DETECT_FIELDS,true);
    return mapper;
}
```

---

#### 2. @JsonIgnore

示例：
我们修改一下`House`类·：
```Java
public class House {
    public int price;
    @JsonIgnore
    private String address;

    public House() {}

    public House(int price, String address) {
        this.price = price;
        this.address = address;
    }
}

```
测试：
```Java
public static void main(String[] args) throws JsonProcessingException {
    ObjectMapper mapper = getObjectMapper();
    String json = mapper.writeValueAsString(getPerson());
    System.out.println(json);
}

public static Person getPerson() {
    House beijing = new House(300,"beijing");
    House shanghai = new House(400, "shanghai");

    return new Person("zed",24, Arrays.asList(beijing,shanghai));
}

public static ObjectMapper getObjectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    mapper.setVisibility(PropertyAccessor.FIELD, JsonAutoDetect.Visibility.ANY) // auto-detect all member fields
            .setVisibility(PropertyAccessor.GETTER, JsonAutoDetect.Visibility.NONE) // but only public getters
            .setVisibility(PropertyAccessor.IS_GETTER, JsonAutoDetect.Visibility.NONE); // and none of "is-setters"

    return mapper;
}
```
OutPut:
```Java
// 只有price， 没有Address。
// 即使我们设置了全局可见。
{"name":"zed","age":24,"houses":[{"price":300},{"price":400}]}

```
当`@JsonIgnore`不管注解在`getters`上还是`setters`上都会忽略对应的属性。



---

#### 3. @JsonProperty
作用在字段或方法上，用来对属性的序列化/反序列化，可以用来避免遗漏属性，同时提供对属性名称重命名.
示例：
我们修改一下`House`类：
```Java
public class House {
    public int price;
    @JsonProperty("destination")
    private String address;

    public House() {}

    public House(int price, String address) {
        this.price = price;
        this.address = address;
    }
}

```
测试：
```Java
public static void main(String[] args) throws JsonProcessingException {
    ObjectMapper mapper = getObjectMapper();
    String json = mapper.writeValueAsString(getPerson());
    System.out.println(json);
}

public static Person getPerson() {
    House beijing = new House(300,"beijing");
    House shanghai = new House(400, "shanghai");

    return new Person("zed",24, Arrays.asList(beijing,shanghai));
}

public static ObjectMapper getObjectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    return mapper;
}
```
OutPut:
```Java
{"name":"zed","age":24,"houses":[{"price":300,"destination":"beijing"},{"price":400,"destination":"shanghai"}]}
```

---


#### 4. @JsonIgnoreProperties
作用在类上，用来说明有些属性在序列化/反序列化时需要忽略掉，可以将它看做是`@JsonIgnore`的批量操作，但它的功能比`@JsonIgnore`要强，比如一个类是代理类，我们无法将将`@JsonIgnore`标记在属性或方法上，此时便可用`@JsonIgnoreProperties`标注在类声明上，它还有一个重要的功能是作用在反序列化时解析字段时过滤一些未知的属性，否则通常情况下解析到我们定义的类不认识的属性便会抛出异常。

示例：
我们修改一下House：
```Java
//注意，@JsonIgnoreProperties("address")不行。
@JsonIgnoreProperties("destination")
public class House {
    public int price;
    @JsonProperty("destination")
    private String address;

    public House() {}

    public House(int price, String address) {
        this.price = price;
        this.address = address;
    }
}

```
测试：
```Java
public static void main(String[] args) throws JsonProcessingException {
    ObjectMapper mapper = getObjectMapper();
    String json = mapper.writeValueAsString(getPerson());
    System.out.println(json);
}

public static Person getPerson() {
    House beijing = new House(300,"beijing");
    House shanghai = new House(400, "shanghai");

    return new Person("zed",24, Arrays.asList(beijing,shanghai));
}

public static ObjectMapper getObjectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    return mapper;
}
```
OutPut:
```Java
{"name":"zed","age":24,"houses":[{"price":300},{"price":400}]}
```
也可以注明过滤掉未知的属性如`@JsonIgnoreProperties(ignoreUnknown=true)`。


---


#### 5. @JsonUnwrapped
作用在属性字段或方法上，用来将子`JSON`对象的属性添加到封闭的`JSON`对象。

说起来比较难懂，看个例子就很清楚了：

示例：
我们修改一下Person：
```Java
public class Person {
    private String name;
    private int age;
    private House houses;

    public Person(String name, int age, House houses) {
        this.name = name;
        this.age = age;
        this.houses = houses;
    }

    public House getHouses() { return houses; }
    public void setHouses(House houses) { this.houses = houses; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
}

```
测试：
```Java
public static void main(String[] args) throws JsonProcessingException {
    ObjectMapper mapper = getObjectMapper();
    String json = mapper.writeValueAsString(getPerson());
    System.out.println(json);
}

public static Person getPerson() {
    House shanghai = new House(400, "shanghai");
    return new Person("zed",24, shanghai);
}

public static ObjectMapper getObjectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    return mapper;
}

```
OutPut:
```Java
{"name":"zed","age":24,"houses":{"price":400,"destination":"shanghai"}}
```

我们给`House`属性加上`@JsonUnwrapped`注解：
```Java
public class Person {
    private String name;
    private int age;
    @JsonUnwrapped
    private House houses;

    public Person(String name, int age, House houses) {
        this.name = name;
        this.age = age;
        this.houses = houses;
    }

    public House getHouses() { return houses; }
    public void setHouses(House houses) { this.houses = houses; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
}

```

OutPut:
```Java
{"name":"zed","age":24,"price":400,"destination":"shanghai"}
```
在2.0+版本中`@JsonUnwrapped`添加了`prefix`和`suffix`属性，用来对字段添加前后缀，这在有关属性分组上比较有用。


---

#### 6. @JsonIdentityInfo
2.0+版本新注解，作用于类或属性上，被用来在序列化/反序列化时为该对象或字段添加一个对象识别码，通常是用来解决循环嵌套的问题，比如数据库中的多对多关系，通过配置属性`generator`来确定识别码生成的方式，有简单的，配置属性`property`来确定识别码的名称，识别码名称没有限制。

- 对象识别码可以是虚拟的，即存在在`JSON`中，但不是`POJO`的一部分，这种情况下我们可以如此使用注解：
  ```Java
  @JsonIdentityInfo(generator =
    ObjectIdGenerators.IntSequenceGenerator.class,property = "@id")
  ```
- 对象识别码也可以是真实存在的，即以对象的属性为识别码，通常这种情况下我们一般以`id`属性为识别码，可以这么使用注解：
  ```Java
  @JsonIdentityInfo(generator =
    ObjectIdGenerators.PropertyGenerator.class,property = "id")
  ```

示例：
```Java
@Test
public void jsonIdentityInfo() throws Exception {
	Parent parent = new Parent();
	parent.setName("jack");
	Child child = new Child();
	child.setName("mike");
	Child[] children = new Child[]{child};
	parent.setChildren(children);
	child.setParent(parent);
	ObjectMapper objectMapper = new ObjectMapper();
	String jsonStr = objectMapper.writeValueAsString(parent);
	Assert.assertEquals("{\"@id\":1,\"name\":\"jack\",
          \"children\":[{\"name\":\"mike\",\"parent\":1}]}",jsonStr);
}

@JsonIdentityInfo(generator = ObjectIdGenerators.IntSequenceGenerator.class,property = "@id")
public static class Parent{
	private String name;
	private Child[] children;

	//getters、setters省略
}

public static class Child{
	private String name;
	private Parent parent;

	//getters、setters省略
}
```
---

#### 7. @JsonNaming
jackson 2.1+版本的注解，作用于类或方法，注意这个注解是在`jackson-databind`包中而不是在`jackson-annotations`包里，它可以让你定制属性命名策略，作用和前面提到的`@JsonProperty`的重命名属性名称相同。

可以用`@JsonProperty`，可是如果`POJO`里有很多属性，给每个属性都要加上`@JsonProperty`是多么繁重的工作，这里就需要用到`@JsonNaming`了，它不仅能制定统一的命名规则，还能任意按自己想要的方式定制。
```Java
@JsonNaming(PropertyNamingStrategy.LowerCaseWithUnderscoresStrategy.class)
public class TestPOJO {

    private String inReplyToUserId;

    public String getInReplyToUserId() {
        return inReplyToUserId;
    }

    public void setInReplyToUserId(String inReplyToUserId) {
        this.inReplyToUserId = inReplyToUserId;
    }
}
```
`@JsonNaming`使用了`jackson`已经实现的`PropertyNamingStrategy.LowerCaseWithUnderscoresStrategy`，它可以将大写转换为小写并添加下划线。你可以自定义，必须继承类`PropertyNamingStrategy`，建议继承`PropertyNamingStrategyBase`.

如果你想让自己定制的策略对所有解析都实现，除了对每个具体的实体类对应的位置加上`@JsonNaming`外你还可以如下做全局配置.
```Java
public static ObjectMapper getObjectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    mapper.setPropertyNamingStrategy(new MyPropertyNamingStrategy());
    return mapper;
}
```
---
### 多态类型处理
jackson允许配置多态类型处理，当进行反序列话时，`JSON`数据匹配的对象可能有多个子类型，为了正确的读取对象的类型，我们需要添加一些类型信息。可以通过下面几个注解来实现。

---
#### 1. @JsonTypeInfo
作用于类/接口，被用来开启多态类型处理，对基类/接口和子类/实现类都有效。

```Java
@JsonTypeInfo(use = JsonTypeInfo.Id.CLASS)
```

**use** :定义使用哪一种类型识别码，它有下面几个可选值：
1. `JsonTypeInfo.Id.CLASS`：使用完全限定类名做识别
2. `JsonTypeInfo.Id.MINIMAL_CLASS`：若基类和子类在同一包类，使用类名(忽略包名)作为识别码
3. `JsonTypeInfo.Id.NAME`：一个合乎逻辑的指定名称
4. `JsonTypeInfo.Id.CUSTOM`：自定义识别码，由`@JsonTypeIdResolver`对应，稍后解释
5. `JsonTypeInfo.Id.NONE`：不使用识别码

**include** (可选):指定识别码是如何被包含进去的，它有下面几个可选值：
1. `JsonTypeInfo.As.PROPERTY`：作为数据的兄弟属性
2. `JsonTypeInfo.As.EXISTING_PROPERTY`：作为`POJO`中已经存在的属性
3. `JsonTypeInfo.As.EXTERNAL_PROPERTY`：作为扩展属性
4. `JsonTypeInfo.As.WRAPPER_OBJECT`：作为一个包装的对象
5. `JsonTypeInfo.As.WRAPPER_ARRAY`：作为一个包装的数组

**property** (可选):制定识别码的属性名称
此属性只有当 **use** 为:
- `JsonTypeInfo.Id.CLASS`（若不指定`property`则默认为`@class`）
- `JsonTypeInfo.Id.MINIMAL_CLASS`(若不指定`property`则默认为`@c`)
- `JsonTypeInfo.Id.NAME`(若不指定`property`默认为`@type`)
且`include`为
- `JsonTypeInfo.As.PROPERTY`
- `JsonTypeInfo.As.EXISTING_PROPERTY`
- `JsonTypeInfo.As.EXTERNAL_PROPERTY`
时才有效

---

#### 2. @JsonSubTypes
作用于类/接口，用来列出给定类的子类，只有当子类类型无法被检测到时才会使用它。

一般是配合`@JsonTypeInfo`在基类上使用。
```Java
@JsonTypeInfo(use = JsonTypeInfo.Id.CLASS)
@JsonSubTypes({
        @JsonSubTypes.Type(value=Person.class,name = "com.learn.Person"),
        @JsonSubTypes.Type(value=PersonSub1.class,name = "com.learn.PersonSub1")
})
```
`@JsonSubTypes`的值是一个`@JsonSubTypes.Type[]`数组，里面枚举了多态类型(`value`对应类)和类型的标识符值(`name`对应`@JsonTypeInfo`中的`property`标识名称的值，此为可选值，若不制定需由`@JsonTypeName`在子类上制定)。

---

以上两个基本适用到大多数情况。

### 用于序列化和反序列化的注解类。

#### 1. @JsonSerialize/@JsonDeserialize
作用于方法和字段上，通过 `using(JsonSerializer)`和`using(JsonDeserializer)`来指定序列化和反序列化的实现.

###### 1.序列化和反序列化转换
通常我们在需要自定义序列化和反序列化时会用到，比如下面的例子中的日期转换.

```Java

class Person {
    private String name;
    @JsonSerialize(using = MyDateSerializer.class)
    @JsonDeserialize(using = MyDateDeserializer.class)
    private Date birthday;

    //getters、setters省略
}


class MyDateSerializer extends JsonSerializer<Date> {
    @Override
    public void serialize(Date value, JsonGenerator jgen, SerializerProvider provider)
            throws IOException, JsonProcessingException {
        DateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        String dateStr = dateFormat.format(value);
        jgen.writeString(dateStr);
    }
}


class MyDateDeserializer extends JsonDeserializer<Date> {
    @Override
    public Date deserialize(JsonParser jp, DeserializationContext ctxt)
            throws IOException, JsonProcessingException {
        String value = jp.getValueAsString();
        DateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        try {
            return dateFormat.parse(value);
        } catch (ParseException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

上面的例子中自定义了日期的序列化和反序列化方式，可以将`Date`和指定日期格式字符串之间相互转换。

###### 2. 实现多态类型转换
也可以通过使用`as(JsonSerializer)`和`as(JsonDeserializer)`来实现多态类型转换，上面我们有提到多态类型处理时可以使用`@JsonTypeInfo`实现，还有一种比较简便的方式就是使用`@JsonSerialize`和`@JsonDeserialize`指定`as`的子类类型，注意这里必须指定为子类类型才可以实现替换运行时的类型。
```Java
    public static class Person {
        private String name;
        @JsonSerialize(as = Sub1.class)
        @JsonDeserialize(as = Sub1.class)
        private MyIn sub1;
        @JsonSerialize(as = Sub2.class)
        @JsonDeserialize(as = Sub2.class)
        private MyIn sub2;
        //getters、setters省略
    }
    public static class MyIn {
        private int id;
        //getters、setters省略
    }
    public static class Sub1 extends MyIn {
        private String name;
        //getters、setters省略
    }
    public static class Sub2 extends MyIn {
        private int age;
        //getters、setters省略
    }
```
上面例子中通过`as`来指定了需要替换实际运行时类型的子类，实际上上面例子中序列化时是可以不使用`@JsonSerialize(as = Sub1.class)`的，因为`jackson`可以自动将`POJO`转换为对应的`JSON`，而反序列化时由于无法自动检索匹配类型必须要指定`@JsonDeserialize(as = Sub1.class)`方可实现。

###### 3. 过滤掉空的属性或有默认值的属性
最后`@JsonSerialize`可以配置`include`属性来指定序列化时被注解的属性被包含的方式，默认总是被包含进来，但是可以过滤掉空的属性或有默认值的属性，举个简单的过滤空属性的例子如下:

```Java
@Test
public void jsonSerializeAndDeSerialize() throws Exception {
    TestPOJO testPOJO = new TestPOJO();
    testPOJO.setName("");
    ObjectMapper objectMapper = new ObjectMapper();
    String jsonStr = objectMapper.writeValueAsString(testPOJO);
    Assert.assertEquals("{}",jsonStr);
}

public static class TestPOJO{
    @JsonSerialize(include = JsonSerialize.Inclusion.NON_EMPTY)
    private String name;

    //getters、setters省略
}

```
---

#### 2. @JsonPropertyOrder
作用在类上，被用来指明当序列化时需要对属性做排序
`@JsonPropertyOrder`有个方法是`alphabetic`：布尔类型，表示是否采用字母拼音顺序排序，默认是为`false`，即不排序。

也可以通过配置来使之生效：
```Java
objectMapper.configure(MapperFeature.SORT_PROPERTIES_ALPHABETICALLY,true);
```

#### 3. @JsonView
视图模板，作用于方法和属性上，用来指定哪些属性可以被包含在`JSON`视图中，在前面我们知道已经有`@JsonIgnore`和`@JsonIgnoreProperties`可以排除过滤掉不需要序列化的属性，可是如果一个`POJO`中有上百个属性，比如订单类、商品详情类这种属性超多，而我们可能只需要概要简单信息即序列化时只想输出其中几个或10几个属性，此时使用`@JsonIgnore`和`@JsonIgnoreProperties`就显得非常繁琐，而使用`@JsonView`便会非常方便，只许在你想要输出的属性(或对应的`getter`)上添加`@JsonView`即可，

#### 4. @JsonAnySetter
作用于方法，在反序列化时用来处理遇到未知的属性的时候调用，在本文前面我们知道可以通过注解`@JsonIgnoreProperties(ignoreUnknown=true)`来过滤未知的属性，但是如果需要这些未知的属性该如何是好?那么`@JsonAnySetter`就可以派上用场了，它通常会和`map`属性配合使用用来保存未知的属性。
```Java
public static class Person{
    private String name;
    private Map other = new HashMap();
    @JsonAnySetter
    public void set(String name,Object value) {
        other.put(name,value);
    }
    //getters、setters省略
}
```

#### 5. @JsonCreator
作用于方法，通常用来标注构造方法或静态工厂方法上，使用该方法来构建实例，默认的是使用无参的构造方法，通常是和`@JsonProperty`或`@JacksonInject`配合使用。
 ```Java
 public static class TestPOJO{
     private String name;
     private int age;

     @JsonCreator
     public TestPOJO(@JsonProperty("full_name") String name,
                     @JsonProperty("age") int age){
         this.name = name;
         this.age = age;
     }
     public String getName() {
         return name;
     }
     public int getAge() {
         return age;
     }
 }
 ```
> 上面是在构造方法上标注了@JsonCreator，同样也可以标注在静态工厂方法上.

还有一种构造方式成为授权式构造器，也是我们平常比较常用到的，这个构造器只有一个参数，且不能使用`@JsonProperty`。
```Java
public static class TestPOJO{
    private String name;
    private int age;
    @JsonCreator
    public TestPOJO(Map map){
        this.name = (String)map.get("full_name");
        this.age = (Integer)map.get("age");
    }
    public String getName() {
        return name;
    }
    public int getAge() {
        return age;
    }
}
```
#### 6. @JacksonInject
作用于属性、方法、构造参数上，被用来反序列化时标记已经被注入的属性.
```Java
@Test
public void jacksonInject() throws Exception {
    ObjectMapper objectMapper = new ObjectMapper();
    String jsonStr = "{\"age\":12}";
    InjectableValues inject = new InjectableValues.Std().addValue("name", "myName");
    Person person = objectMapper.reader(Person.class).with(inject).readValue(jsonStr);
    Assert.assertEquals("myName", person.getName());
    Assert.assertEquals(12, person.getAge());
}

public static class Person {
    @JacksonInject("name")
    private String name;
    private int age;

    //getters、setters省略
}
```
#### 7. @JsonPOJOBuilder
作用于类，用来标注如何定制构建对象，使用的是`builder`模式来构建，比如`Value v = new ValueBuilder().withX(3).withY(4).build()`;这种就是`builder`模式来构建对象，通常会和`@JsonDeserialize.builder`来配合使用。
```Java
@Test
public void jacksonInject() throws Exception {
    ObjectMapper objectMapper = new ObjectMapper();
    String jsonStr = "{\"age\":12}";
    InjectableValues inject = new InjectableValues.Std().addValue("name", "myName");
    Person person = objectMapper.reader(Person.class).with(inject).readValue(jsonStr);
    Assert.assertEquals("myName", person.getName());
    Assert.assertEquals(12, person.getAge());
}


@JsonDeserialize(builder = PersonBuilder.class)
public static class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }
}

@JsonPOJOBuilder(buildMethodName = "create", withPrefix = "with")
public static class PersonBuilder {
    private String name;
    private int age;

    public PersonBuilder withName(String name) {
        this.name = name;
        return this;
    }

    public PersonBuilder withAge(int age) {
        this.age = age;
        return this;
    }

    public Person create() {
        return new Person(name, age);
    }
}
```
在`PersonBuilder`上有`@JsonPOJOBuilder`注解，表示所有的参数传递方法都是以`with`开头，最终构建好的对象是通过`create`方法来获得，而在`Person`上使用了`@JsonDeserializer`，告诉我们在反序列化的时候我们使用的是`PersonBuilder`来构建此对象的。






## 三 配置SerializationFeature
> 以下的示例除去`getObjectMapper()`都相同。
```Java
public static void main(String[] args) throws JsonProcessingException {
    ObjectMapper mapper = getObjectMapper();
    String json = mapper.writeValueAsString(getPerson());
    System.out.println(json);
}

public static Person getPerson() {
    House beijing = new House(300,"beijing");
    House shanghai = new House(400, "shanghai");

    return new Person("zed",24, Arrays.asList(beijing,shanghai));
}

public static ObjectMapper getObjectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    return mapper;
}
```

#### SerializationFeature.WRAP_ROOT_VALUE
是否环绕根元素，默认`false`，如果为`true`，则默认以类名作为根元素，你也可以通过`@JsonRootName`来自定义根元素名称
```Java
public static ObjectMapper getObjectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    mapper.configure(SerializationFeature.WRAP_ROOT_VALUE,true);
    return mapper;
}
```
OutPut:
```Java
// 有Person 有houses。
{"Person":{"name":"zed","age":24,"houses":[{"price":300,"address":"beijing"},{"price":400,"address":"shanghai"}]}}
```
#### SerializationFeature.INDENT_OUTPUT
是否缩放排列输出，默认`false`，有些场合为了便于排版阅读则需要对输出做缩放排列
```Java
public static ObjectMapper getObjectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    mapper.configure(SerializationFeature.INDENT_OUTPUT,true);
    return mapper;
}

```
OutPut:
```Json
{
  "name" : "zed",
  "age" : 24,
  "houses" : [ {
    "price" : 300,
    "address" : "beijing"
  }, {
    "price" : 400,
    "address" : "shanghai"
  } ]
}
```


#### SerializationFeature.ORDER_MAP_ENTRIES_BY_KEYS
序列化Map时对key进行排序操作，默认`false`.




#### SerializationFeature.WRITE_CHAR_ARRAYS_AS_JSON_ARRAYS
序列化`char[]`时以`json`数组输出，默认`false`。

需要注意的是对于第二种通过配置`SerializationConfig`和`DeserializationConfig`方式只能启动/禁止自动检测，无法修改我们所需的可见级别。

有时候对每个实例进行可见级别的注解可能会非常麻烦，这时候我们需要配置一个全局的可见级别，通过`objectMapper.setVisibilityChecker()`来实现，默认的`VisibilityChecker`实现类为`VisibilityChecker.Std`，这样可以满足实现复杂场景下的基础配置。
```Java
public static ObjectMapper getObjectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    mapper.setVisibility(PropertyAccessor.FIELD, JsonAutoDetect.Visibility.ANY) // auto-detect all member fields
            .setVisibility(PropertyAccessor.GETTER, JsonAutoDetect.Visibility.NONE) // but only public getters
            .setVisibility(PropertyAccessor.IS_GETTER, JsonAutoDetect.Visibility.NONE); // and none of "is-setters"

    return mapper;
}
```
## 四 集合的操作。

```Java
//思考:为什么需要指定类型？(类型擦除)
Map<String, ResultValue> results = mapper.readValue(jsonSource,
        new TypeReference<Map<String, ResultValue>>() {});
```

---

## 五 树模型
虽然看起来处理的很方便，但是某些时候会有一些很麻烦的情况，这时候可以考虑使用树模型:

```Java
//如果结果可能是Object或者是Array，那可以使用JsonNode;
//如果你知道是Object，你可以直接强转成ObjectNode;如果你知道是Array，你可以直接强转成ArrayNode;
String json = ...;
ObjectNode root = (ObjectNode) mapper.readTree(json);
String name = root.get("name").asText();
int age = root.get("age").asInt();

// 还可以修改这个树，然后再输出成json字符串
root.with("other").put("type", "student");
String json = mapper.writeValueAsString(root);

```
如果`json`类型太过动态，不适合反序列化成对象的时候，树模型比数据绑定更合适。
---

感谢：
[jackson annotations注解详解][1]






[1]:https://blog.csdn.net/sdyy321/article/details/40298081
