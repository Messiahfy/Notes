## 一、基本使用

#### 基本数据类型
```
1. 序列化（即转为字符串）
Gson gson = new Gson();
gson.toJson(1);            // ==> 1  返回的字符串字面量为"1"，打印则没有引号
gson.toJson("abcd");       // ==> "abcd"  打印出来带引号，返回的原本字符串字面量为"\"abcd\""
gson.toJson(new Long(10)); // ==> 10
int[] values = { 1 };
gson.toJson(values);       // ==> [1]

2. 反序列化（从字符串转为变量）    从字符串转为变量需要告知要转为的变量类型
int one = gson.fromJson("1", int.class);
Integer one = gson.fromJson("1", Integer.class);
Long one = gson.fromJson("1", Long.class);
Boolean false = gson.fromJson("false", Boolean.class);
String str = gson.fromJson("\"abc\"", String.class);
String[] anotherStr = gson.fromJson("[\"abc\"]", String[].class);
```

#### 对象类型
```
class BagOfPrimitives {
  private int value1 = 1;
  private String value2 = "abc";
  private transient int value3 = 3;
  BagOfPrimitives() {
    // 无参构造函数
  }
}

// 序列化，对象转为字符串
BagOfPrimitives obj = new BagOfPrimitives();
Gson gson = new Gson();
String json = gson.toJson(obj);  

// ==> json is {"value1":1,"value2":"abc"}

// 反序列化
BagOfPrimitives obj2 = gson.fromJson(json, BagOfPrimitives.class);
// ==> obj2 is just like obj
```
> **注意**：不能序列化循环引用的对象，因为会导致无限递归。

#### 关于对象的细节
* 建议使用私有字段
* 不需要对要序列化或反序列化的字段加注解， 默认情况下包含当前类（以及所有超类）中的所有字段
* 如果字段标记为transient，（默认）则忽略该字段，并且不包括在JSON序列化或反序列化中。
* 关于空值：序列化时省略null字段；反序列化时，JSON字符串中没有的字段在对象中设为默认值，对象类型为null，数字类型为零，布尔值为false。
* 如果字段是synthetic，则会被忽略，并且不包含在JSON序列化或反序列化中。
* 内部类、匿名类和局部类中的外部类对应的字段将被忽略，不包括在序列化或反序列化中。（内部类等的对象都含有外部类的引用）

> 不能把一个JSON字符串反序列化为一个内部类对象，除非是静态内部类

#### 数组例子
```
Gson gson = new Gson();
int[] ints = {1, 2, 3, 4, 5};
String[] strings = {"abc", "def", "ghi"};

// 序列化
gson.toJson(ints);     // ==> [1,2,3,4,5]
gson.toJson(strings);  // ==> ["abc", "def", "ghi"]

// 反序列化
int[] ints2 = gson.fromJson("[1,2,3,4,5]", int[].class); 
// ==> ints2 的值将和ints一致
```
> 支持任意复杂类型的多维数组

#### 集合例子
```
Gson gson = new Gson();
Collection<Integer> ints = Lists.immutableList(1,2,3,4,5);

// 序列化
String json = gson.toJson(ints);  // ==> json is [1,2,3,4,5]

// 反序列化
Type collectionType = new TypeToken<Collection<Integer>>(){}.getType();
Collection<Integer> ints2 = gson.fromJson(json, collectionType);
// ==> ints2 is same as ints
```
> 序列化集合成JSON字符串没有问题，但从JSON反序列化为集合时，则必须用上面这个较丑陋的`TypeToken<Collection<Integer>>(){}.getType()`来指定集合类型。事实上，只要是泛型就都要使用这个方式来反序列化。

#### 序列化和反序列化泛型类型
```
class Foo<T> {
  T value;
}
Gson gson = new Gson();
Foo<Bar> foo = new Foo<Bar>();

Type fooType = new TypeToken<Foo<Bar>>() {}.getType();
gson.toJson(foo, fooType);

gson.fromJson(json, fooType);
```

---------------------------------------------------------------

#### 使用任意类型的对象序列化和反序列化集合
有时要处理包含混合类型的JSON数组，例如：['hello',5,{name:'GREETINGS',source:'guest'}]
相当于如下`Collection `
```
Collection collection = new ArrayList();
collection.add("hello");
collection.add(5);
collection.add(new Event("GREETINGS", "guest"));
```
Event类定义如下：
```
class Event {
  private String name;
  private String source;
  private Event(String name, String source) {
    this.name = name;
    this.source = source;
  }
}
```
可以使用`Gson`序列化集合，而无需执行任何特定操作：`toJson(collection)`将写出所需的输出。

但是反序列化则需要我们做一些操作，**由于是混合类型的集合**，所以**不能**像前面简单的使用`TypeToken`来解决。这里我们有三个选项：
1. 使用`Gson`中的`JsonArray`和`JsonParser`来解决（用得较少）。[示例](https://github.com/google/gson/blob/master/extras/src/main/java/com/google/gson/extras/examples/rawcollections/RawCollectionsExample.java)
2. 为`Collection.class`注册一个**type adapter**，它查看每个数组成员并将它们映射到适当的对象。 这种方法的缺点是它会搞砸`Gson`中其他集合类型的反序列化。
3. 为·MyCollectionMemberType·注册一个**type adapter**，并用`Collection <MyCollectionMemberType>`来使用`fromJson()`。

仅当数组显示为顶级元素或者您可以将包含集合的字段类型更改为Collection <MyCollectionMemberType>类型时，此方法才可用。？？？？？？？？？？？？？？？？

> 这些就属于自定义序列化和反序列化了，下面是关于这方面的具体使用

## 二、自定义序列化和反序列化

#### TypeAdapter以及JsonSerializer与JsonDeserializer
参考[你真的会用Gson吗？](https://www.jianshu.com/p/3108f1e44155)


## 别名
```
@SerializedName("email_address")
public String emailAddress;
```

多个备选
```
@SerializedName(value = "emailAddress", alternate = {"email", "email_address"})
public String emailAddress;
```
如果JSON字符串中同时出现emailAddress、email、email_address，则以最后出现的为准