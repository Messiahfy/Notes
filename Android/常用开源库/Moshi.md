PolymorphicJsonAdapterFactory 多态


@JsonClass(generateAdapter = true)
可以使用kapt或ksp生成代码，比如使用ksp会生成文件 /build/generated/ksp/debug/kotlin/xxx.xxx/XXJsonAdapter

Factory BUILT_IN_FACTORIES StandardJsonAdapters 会反射创建生成的Adapter实例

ClassJsonAdapter用于兜底，比如没有配置@JsonClass生成代码，就直接反射（但需要引入kotlin-reflect库）


```
val adapter = moshi.adapter(Foo::class.java)
val foo = adapter.fromJson(jsonString)
```

寻找adapter的核心代码：
```
for (int i = 0, size = factories.size(); i < size; i++) {
  JsonAdapter<T> result = (JsonAdapter<T>) factories.get(i).create(type, annotations, this);
  if (result == null) continue

  // Success! Notify the LookupChain so it is cached and can be used by re-entrant calls.
  lookupChain.adapterFound(result);
  success = true;
  return result;
}
```
遍历注册的`List<JsonAdapter.Factory>`，返回非空即可用

而`factories`默认会注册Moshi自带的BUILT_IN_FACTORIES，和用户注册的。

```
  static {
    BUILT_IN_FACTORIES.add(StandardJsonAdapters.FACTORY);
    BUILT_IN_FACTORIES.add(CollectionJsonAdapter.FACTORY);
    BUILT_IN_FACTORIES.add(MapJsonAdapter.FACTORY);
    BUILT_IN_FACTORIES.add(ArrayJsonAdapter.FACTORY);
    BUILT_IN_FACTORIES.add(ClassJsonAdapter.FACTORY);
  }
```

没有自己注册的话，会在默认的JsonAdapter.Factory里面选。比如`StandardJsonAdapters.FACTORY`：
```
final class StandardJsonAdapters {
    public static final JsonAdapter.Factory FACTORY = new JsonAdapter.Factory() {
        public JsonAdapter<?> create(Type type, Set<? extends Annotation> annotations, Moshi moshi) {
            if (!annotations.isEmpty()) {
                return null;
            } else if (type == Boolean.TYPE) {
                return StandardJsonAdapters.BOOLEAN_JSON_ADAPTER;
            } else if (type == Byte.TYPE) {
                return StandardJsonAdapters.BYTE_JSON_ADAPTER;
            } else if (type == Character.TYPE) {
                return StandardJsonAdapters.CHARACTER_JSON_ADAPTER;
            } else if (type == Double.TYPE) {
                return StandardJsonAdapters.DOUBLE_JSON_ADAPTER;
            } else if (type == Float.TYPE) {
                return StandardJsonAdapters.FLOAT_JSON_ADAPTER;
            } else if (type == Integer.TYPE) {
                return StandardJsonAdapters.INTEGER_JSON_ADAPTER;
            } else if (type == Long.TYPE) {
                return StandardJsonAdapters.LONG_JSON_ADAPTER;
            } else if (type == Short.TYPE) {
                return StandardJsonAdapters.SHORT_JSON_ADAPTER;
            } else if (type == Boolean.class) {
                return StandardJsonAdapters.BOOLEAN_JSON_ADAPTER.nullSafe();
            } else if (type == Byte.class) {
                return StandardJsonAdapters.BYTE_JSON_ADAPTER.nullSafe();
            } else if (type == Character.class) {
                return StandardJsonAdapters.CHARACTER_JSON_ADAPTER.nullSafe();
            } else if (type == Double.class) {
                return StandardJsonAdapters.DOUBLE_JSON_ADAPTER.nullSafe();
            } else if (type == Float.class) {
                return StandardJsonAdapters.FLOAT_JSON_ADAPTER.nullSafe();
            } else if (type == Integer.class) {
                return StandardJsonAdapters.INTEGER_JSON_ADAPTER.nullSafe();
            } else if (type == Long.class) {
                return StandardJsonAdapters.LONG_JSON_ADAPTER.nullSafe();
            } else if (type == Short.class) {
                return StandardJsonAdapters.SHORT_JSON_ADAPTER.nullSafe();
            } else if (type == String.class) {
                return StandardJsonAdapters.STRING_JSON_ADAPTER.nullSafe();
            } else if (type == Object.class) {
                return (new ObjectJsonAdapter(moshi)).nullSafe();
            } else {
                Class<?> rawType = Types.getRawType(type);
                JsonAdapter<?> generatedAdapter = Util.generatedAdapter(moshi, type, rawType);
                if (generatedAdapter != null) {
                    return generatedAdapter;
                } else {
                    return rawType.isEnum() ? (new EnumJsonAdapter(rawType)).nullSafe() : null;
                }
            }
        }
    };

    static final JsonAdapter<Boolean> BOOLEAN_JSON_ADAPTER = new JsonAdapter<Boolean>() {
        public Boolean fromJson(JsonReader reader) throws IOException {
            return reader.nextBoolean();
        }

        public void toJson(JsonWriter writer, Boolean value) throws IOException {
            writer.value(value);
        }

        public String toString() {
            return "JsonAdapter(Boolean)";
        }
    };

    //......

}
```

根据传入的 type ，返回对应的JsonAdapter。

比如 Boolean 类型：
```
val moshi = Moshi.Builder().build()
val adapter = moshi.adapter(java.lang.Boolean.TYPE)
val text = adapter.toJson(true) // 打印就是 true，非对象类型没有大括号等符号
```

如果是自定义的类型，可以在上面`StandardJsonAdapters.FACTORY`的代码看到，create方法最下面，会查找该自定义类型是否使用了`@JsonClass`注解，有的话就会反射查找生成的Adapter。

如果是集合、Map、数组，则会使用`BUILT_IN_FACTORIES`中的`CollectionJsonAdapter`、`MapJsonAdapter.FACTORY`和`ArrayJsonAdapter.FACTORY`。最终才使用`ClassJsonAdapter.Factory`来反射每个字段兜底（可以使用`@Json`）


@JsonClass也会通过ksp/kapt在resources/META-INF/proguard目录中生成proguard文件


```
@JsonClass(generateAdapter = true)
class Person(
    val id: Long,
    val name: String,
    val cars: List<Car>,
)

@JsonClass(generateAdapter = true)
class Car(
    val brand: String,
    val color: String,
)
```
生成代码`JsonAdapter`：
```
public class PersonJsonAdapter(
  moshi: Moshi,
) : JsonAdapter<Person>() {
    // Persion的三个字段
  private val options: JsonReader.Options = JsonReader.Options.of("id", "name", "cars")

    // 用到了 long，所以要用 longAdapter 来处理 id 字段
  private val longAdapter: JsonAdapter<Long> = moshi.adapter(Long::class.java, emptySet(), "id")
    // 用到了 String，所以要用 stringAdapter 来处理 name 字段
  private val stringAdapter: JsonAdapter<String> = moshi.adapter(String::class.java, emptySet(),
      "name")

    // 用到了 List<Car> ，所有要用 listOfCarAdapter 来处理 cars 字段
  private val listOfCarAdapter: JsonAdapter<List<Car>> =
      moshi.adapter(Types.newParameterizedType(List::class.java, Car::class.java), emptySet(),
      "cars")

  override fun toString(): String = buildString(28) {
      append("GeneratedJsonAdapter(").append("Person").append(')') }

  override fun fromJson(reader: JsonReader): Person {
    var id: Long? = null
    var name: String? = null
    var cars: List<Car>? = null
    reader.beginObject() // 以当前读取位置为一个json对象来读取，即下一个字符为‘{’
    while (reader.hasNext()) {
      when (reader.selectName(options)) { // 找到下一个key是哪个，就选对应的adapter来处理
        0 -> id = longAdapter.fromJson(reader) ?: throw Util.unexpectedNull("id", "id", reader)
        1 -> name = stringAdapter.fromJson(reader) ?: throw Util.unexpectedNull("name", "name",
            reader)
        2 -> cars = listOfCarAdapter.fromJson(reader) ?: throw Util.unexpectedNull("cars", "cars",
            reader)
        -1 -> {
          // Unknown name, skip it.
          reader.skipName()
          reader.skipValue()
        }
      }
    }
    reader.endObject()
    return Person(
        id = id ?: throw Util.missingProperty("id", "id", reader),
        name = name ?: throw Util.missingProperty("name", "name", reader),
        cars = cars ?: throw Util.missingProperty("cars", "cars", reader)
    )
  }

  override fun toJson(writer: JsonWriter, value_: Person?) {
    if (value_ == null) {
      throw NullPointerException("value_ was null! Wrap in .nullSafe() to write nullable values.")
    }
    writer.beginObject()
    writer.name("id")
    longAdapter.toJson(writer, value_.id)
    writer.name("name")
    stringAdapter.toJson(writer, value_.name)
    writer.name("cars")
    listOfCarAdapter.toJson(writer, value_.cars)
    writer.endObject()
  }
}
```

```
public class CarJsonAdapter(
  moshi: Moshi,
) : JsonAdapter<Car>() {
  private val options: JsonReader.Options = JsonReader.Options.of("brand", "color")

  private val stringAdapter: JsonAdapter<String> = moshi.adapter(String::class.java, emptySet(),
      "brand")

  override fun toString(): String = buildString(25) {
      append("GeneratedJsonAdapter(").append("Car").append(')') }

  override fun fromJson(reader: JsonReader): Car {
    var brand: String? = null
    var color: String? = null
    reader.beginObject()
    while (reader.hasNext()) {
      when (reader.selectName(options)) {
        0 -> brand = stringAdapter.fromJson(reader) ?: throw Util.unexpectedNull("brand", "brand",
            reader)
        1 -> color = stringAdapter.fromJson(reader) ?: throw Util.unexpectedNull("color", "color",
            reader)
        -1 -> {
          // Unknown name, skip it.
          reader.skipName()
          reader.skipValue()
        }
      }
    }
    reader.endObject()
    return Car(
        brand = brand ?: throw Util.missingProperty("brand", "brand", reader),
        color = color ?: throw Util.missingProperty("color", "color", reader)
    )
  }

  override fun toJson(writer: JsonWriter, value_: Car?) {
    if (value_ == null) {
      throw NullPointerException("value_ was null! Wrap in .nullSafe() to write nullable values.")
    }
    writer.beginObject()
    writer.name("brand")
    stringAdapter.toJson(writer, value_.brand)
    writer.name("color")
    stringAdapter.toJson(writer, value_.color)
    writer.endObject()
  }
}
```
嵌套类就会使用嵌套的JsonAdapter来处理。最终还是会用到基本类型、String、集合等的Adapter，因为自定义的类的数据最终也都是基本类型和字符串、集合等类型。