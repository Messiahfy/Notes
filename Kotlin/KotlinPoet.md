# KotlinPoet

## 代码和控制流
KotlinPoet的API大量采用不可变对象，Builder模式、链式调用和varargs不定数量参数

KotlinFile  FileSpec
class、interface（包括注解类、枚举类、object类）  TypeSpec
type aliases TypeAliasSpec
properties  PropertySpec
普通函数/构造函数  FunSpec
参数 ParameterSpec
注解 AnnotationSpec


但是方法和构造函数的内容没有模型化，没有表达式、语句、或者语法树节点
而是使用字符串作为代码块，可以使用Koltin的多行字符串
```
val main = FunSpec.builder("main")
  .addCode("""
    |var total = 0
    |for (i in 0..<10) {
    |    total += i
    |}
    |""".trimMargin())
  .build()
```
生成代码：
```
fun main() {
  var total = 0
  for (i in 0..<10) {
    total += i
  }
}
```

还有另外的api，支持新行、大括号和缩进，也可以生成上面的代码
```
val main = FunSpec.builder("main")
  .addStatement("var total = 0")
  .beginControlFlow("for (i in 0..<10)")
  .addStatement("total += i")
  .endControlFlow()
  .build()
```


但上面的例子，循环0到10是固定的，如果想动态生成：
```
private fun computeRange(name: String, from: Int, to: Int, op: String): FunSpec {
  return FunSpec.builder(name)
    .returns(Int::class)
    .addStatement("var result = 1")
    .beginControlFlow("for (i in $from..<$to)")
    .addStatement("result = result $op i")
    .endControlFlow()
    .addStatement("return result")
    .build()
}
```

## 格式规范

### %S 字符串格式化
函数语句 addStatement 支持 %s
```
private fun whatsMyNameYo(name: String): FunSpec {
  return FunSpec.builder(name)
    .returns(String::class)
    .addStatement("return %S", name)
    .build()
}
```
### %P 字符串模板
支持字符串模板
```
val stringWithADollar = "Your total is " + "$" + "amount"
val funSpec = FunSpec.builder("printTotal")
  .returns(String::class)
  .addStatement("return %P", stringWithADollar)
  .build()

// 生成代码
fun printTotal(): String = "Your total is $amount"
```

### %T 用于类型
可以使用Class<>、TypeName、String
```
val today = FunSpec.builder("today")
  .returns(Date::class)
  .addStatement("return %T()", Date::class)
  .build()

val helloWorld = TypeSpec.classBuilder("HelloWorld")
  .addFunction(today)
  .build()

val kotlinFile = FileSpec.builder("com.example.helloworld", "HelloWorld")
  .addType(helloWorld)
  .build()

kotlinFile.writeTo(System.out)

```

可以填上类型构造函数，并且自动添加import
```
package com.example.helloworld

import java.util.Date

class HelloWorld {
  fun today(): Date = Date()
}
```

比如 PropertySpec initializer 如果不用 %T，用的 %S 导致初始化为字符串了
```
val x: Foo = "Foo()"

// 而应该是这样，这种情况要使用%T
val x: Foo = Foo()
```

### %M 用于成员函数和属性
和`%T`类似，但针对的是成员函数和属性
```
val createTaco = MemberName("com.squareup.tacos", "createTaco")
val isVegan = MemberName("com.squareup.tacos", "isVegan")
val file = FileSpec.builder("com.squareup.example", "TacoTest")
  .addFunction(
    FunSpec.builder("main")
      .addStatement("val taco = %M()", createTaco)
      .addStatement("println(taco.%M)", isVegan)
      .build()
  )
  .build()
println(file)
```

```
package com.squareup.example

import com.squareup.tacos.createTaco
import com.squareup.tacos.isVegan

fun main() {
  val taco = createTaco()
  println(taco.isVegan)
}
```

### %N 用于名称
```
val hexDigit = FunSpec.builder("hexDigit")
  .addParameter("i", Int::class)
  .returns(Char::class)
  .addStatement("return (if (i < 10) i + '0'.toInt() else i - 10 + 'a'.toInt()).toChar()")
  .build()

val byteToHex = FunSpec.builder("byteToHex")
  .addParameter("b", Int::class)
  .returns(String::class)
  .addStatement("val result = CharArray(2)")
  .addStatement("result[0] = %N((b ushr 4) and 0xf)", hexDigit)
  .addStatement("result[1] = %N(b and 0xf)", hexDigit)
  .addStatement("return String(result)")
  .build()
```

```
fun byteToHex(b: Int): String {
  val result = CharArray(2)
  result[0] = hexDigit((b ushr 4) and 0xf)
  result[1] = hexDigit(b and 0xf)
  return String(result)
}

fun hexDigit(i: Int): Char {
  return (if (i < 10) i + '0'.toInt() else i - 10 + 'a'.toInt()).toChar()
}
```


可以看源码会把name取出来，放到%N占位的地方
```
private fun argToName(o: Any?) = when (o) {
      is CharSequence -> o.toString()
      is ParameterSpec -> o.name
      is PropertySpec -> o.name
      is FunSpec -> o.name
      is TypeSpec -> o.name!!
      is MemberName -> o.simpleName
      else -> throw IllegalArgumentException("expected name but was $o")
    }
```


### %L 用于字面量
类似`String.format()`使用的%s

```
private fun computeRange(name: String, from: Int, to: Int, op: String): FunSpec {
  return FunSpec.builder(name)
    .returns(Int::class)
    .addStatement("var result = 0")
    .beginControlFlow("for (i in %L..<%L)", from, to)
    .addStatement("result = result %L i", op)
    .endControlFlow()
    .addStatement("return result")
    .build()
}
```
可以用于字符串、基本类型、和少数KotlinPoet types 

### CodeBlock
```
// 按顺序
CodeBlock.builder().add("I ate %L %L", 3, "tacos")

// 按指定位置
CodeBlock.builder().add("I ate %2L %1L", "tacos", 3)

// 按命名
val map = LinkedHashMap<String, Any>()
map += "food" to "tacos"
map += "count" to 3
CodeBlock.builder().addNamed("I ate %count:L %food:L", map)
```

## 函数
```
val flux = FunSpec.builder("flux")
  .addModifiers(KModifier.ABSTRACT, KModifier.PROTECTED)
  .build()

val helloWorld = TypeSpec.classBuilder("HelloWorld")
  .addModifiers(KModifier.ABSTRACT)
  .addFunction(flux)
  .build()

// 生成代码
abstract class HelloWorld {
  protected abstract fun flux()
}
```
还可以配置 参数、注解、返回值、扩展函数receiver 等

### 扩展函数
```
val square = FunSpec.builder("square")
  .receiver(Int::class)
  .returns(Int::class)
  .addStatement("var s = this * this")
  .addStatement("return s")
  .build()

// 生成代码
fun Int.square(): Int {
  val s = this * this
  return s
}
```

默认参数、


这里的`·`号可以让KotlinPoet识别到，避免map和`{`换行，导致语法错误 
```
val funSpec = FunSpec.builder("foo")
  .addStatement("return (100..10000).map·{ number -> number * number }.map·{ number -> number.toString() }.also·{ string -> println(string) }")
  .build()


fun foo() = (100..10000).map { number -> number * number }.map { number ->
  number.toString()
}.also { string -> println(string) }
```

## 构造函数
```
val flux = FunSpec.constructorBuilder()
  .addParameter("greeting", String::class)
  .addStatement("this.%N = %N", "greeting", "greeting")
  .build()

val helloWorld = TypeSpec.classBuilder("HelloWorld")
  .addProperty("greeting", String::class, KModifier.PRIVATE)
  .addFunction(flux)
  .build()


class HelloWorld {
  private val greeting: String

  constructor(greeting: String) {
    this.greeting = greeting
  }
}
```

主构造函数
```
val helloWorld = TypeSpec.classBuilder("HelloWorld")
  .primaryConstructor(flux)
  .addProperty("greeting", String::class, KModifier.PRIVATE)
  .build()
```

主构造函数直接初始化属性
```
val flux = FunSpec.constructorBuilder()
  .addParameter("greeting", String::class)
  .build()

val helloWorld = TypeSpec.classBuilder("HelloWorld")
  .primaryConstructor(flux)
  .addProperty(
    PropertySpec.builder("greeting", String::class)
      .initializer("greeting")
      .addModifiers(KModifier.PRIVATE)
      .build()
  )
  .build()

class HelloWorld(private val greeting: String)
```


## 参数
```
val android = ParameterSpec.builder("android", String::class)
  .defaultValue("\"pie\"")
  .build()

val welcomeOverlords = FunSpec.builder("welcomeOverlords")
  .addParameter(android)
  .addParameter("robot", String::class)
  .build()
```

```
fun welcomeOverlords(android: String = "pie", robot: String) {
}
```

## 属性
```
val android = PropertySpec.builder("android", String::class)
  .addModifiers(KModifier.PRIVATE)
  .build()

val helloWorld = TypeSpec.classBuilder("HelloWorld")
  .addProperty(android)
  .addProperty("robot", String::class, KModifier.PRIVATE)
  .build()
```

```
class HelloWorld {
  private val android: String

  private val robot: String
}
```


默认为`val`，如果想设置为`var`，可以使用`mutable()`

可以给属性的getter/setter设置`inline`
```
val android = PropertySpec.builder("android", String::class)
  .mutable()
  .getter(
    FunSpec.getterBuilder()
      .addModifiers(KModifier.INLINE)
      .addStatement("return %S", "foo")
      .build()
  )
  .build()
```

```
var android: kotlin.String
  inline get() = "foo"
```

## 接口
```
val helloWorld = TypeSpec.interfaceBuilder("HelloWorld")
  .addProperty("buzz", String::class)
  .addFunction(
    FunSpec.builder("beep")
      .addModifiers(KModifier.ABSTRACT)
      .build()
  )
  .build()
```


`fun interface`
```
val helloWorld = TypeSpec.funInterfaceBuilder("HelloWorld")
  .addFunction(
    FunSpec.builder("beep")
      .addModifiers(KModifier.ABSTRACT)
      .build()
  )
  .build()

// Generates...
fun interface HelloWorld {
  fun beep()
}
```

## Object类和companionObject
object 类
```
val helloWorld = TypeSpec.objectBuilder("HelloWorld")
  .addProperty(
    PropertySpec.builder("buzz", String::class)
      .initializer("%S", "buzz")
      .build()
  )
  .addFunction(
    FunSpec.builder("beep")
      .addStatement("println(%S)", "Beep!")
      .build()
  )
  .build()
```

companion object
```
val companion = TypeSpec.companionObjectBuilder()
  .addProperty(
    PropertySpec.builder("buzz", String::class)
      .initializer("%S", "buzz")
      .build()
  )
  .addFunction(
    FunSpec.builder("beep")
      .addStatement("println(%S)", "Beep!")
      .build()
  )
  .build()

val helloWorld = TypeSpec.classBuilder("HelloWorld")
  .addType(companion)
  .build()
```

## 枚举
```
val helloWorld = TypeSpec.enumBuilder("Roshambo")
  .addEnumConstant("ROCK")
  .addEnumConstant("SCISSORS")
  .addEnumConstant("PAPER")
  .build()

// 生成代码
enum class Roshambo {
  ROCK,

  SCISSORS,

  PAPER
}
```

也支持枚举构造函数、枚举匿名类、枚举常量 等特性，具体可查看API

## 匿名内部类
TypeSpec.anonymousClassBuilder()

## 注解
添加注解到方法上
```
val test = FunSpec.builder("testStringEquality")
  .addAnnotation(Test::class)
  .addStatement("assertThat(%1S).isEqualTo(%1S)", "foo")
  .build()

@Test
fun `testStringEquality`() {
  assertThat("foo").isEqualTo("foo")
}
```

设置注解的属性
```
val logRecord = FunSpec.builder("recordEvent")
  .addModifiers(KModifier.ABSTRACT)
  .addAnnotation(
    AnnotationSpec.builder(Headers::class)
      .addMember("accept = %S", "application/json; charset=utf-8")
      .addMember("userAgent = %S", "Square Cash")
      .build()
  )
  .addParameter("logRecord", LogRecord::class)
  .returns(LogReceipt::class)
  .build()

@Headers(
  accept = "application/json; charset=utf-8",
  userAgent = "Square Cash"
)
abstract fun recordEvent(logRecord: LogRecord): LogReceipt
```

其他使用可查看API

## Type Aliases
TypeAliasSpec.builder

## 可调用的引用
ClassName.constructorReference() for constructors

MemberName.reference() for functions and properties

## 泛型


```
// JsonAdapter<Map<String, Any>>

PropertySpec.builder(
    adapterPropertyName,
    JsonAdapter::class.asClassName().parameterizedBy(
        Map::class.asClassName()
            .parameterizedBy(String::class.asClassName(), Any::class.asClassName())
    ),
    KModifier.PRIVATE
)
```

# 互操作
## KSP
可以转换KSP类型到KotlinPoet类型，并且写入KSP CodeGenerator.
```
dependencies {
  implementation("com.squareup:kotlinpoet-ksp:<version>")
}
```

https://square.github.io/kotlinpoet/interop-ksp/