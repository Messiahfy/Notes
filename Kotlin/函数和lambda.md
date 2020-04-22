## 函数
函数的常规声明：
```
fun sum(x: Int, y: Int): Int {
    return x + y
}
```

当函数返回单个表达式时，可以省略花括号并且在 = 符号之后指定代码体即可：
```
fun sum(x: Int, y: Int): Int = x + y

```
当返回值类型可由编译器推断时，显式声明返回类型是可选的：
```
fun sum(x: Int, y: Int) = x + y

```

### 高阶函数
高阶函数是将函数用作参数或返回值的函数。

#### 函数类型
既然要把函数作为参数或者返回值，显然需要声明函数的类型。函数类型的声明表示如下示例：
```
(Int, Int) -> Boolean
```
表示一个传入两个整型值并返回布尔值的函数类型
声明了函数类型，对于函数类型的实例化，有几种方式：

1. 使用函数字面量的代码块：  

使用lambda表达式来实例化声明的函数：
```
val function: (Int, Int) -> Boolean = { a: Int, b: Int -> a == b }
```
使用匿名函数来实例化声明的函数：
```
val function1: (Int, Int) -> Boolean = fun(x: Int, y: Int): Boolean { return x == y }
```
可以知道lambda表达式和匿名函数都属于函数类型的字面量，也就是函数类型的值。

2. 使用已有声明的可调用引用：
* 顶层、局部、成员、扩展函数：::isOdd、 String::toInt
* 顶层、成员、扩展属性：
```
//顶层属性
val x = 1

fun main() {
    println(::x.get())
    println(::x.name) 
}

//成员属性 Book类
Book::name  函数类型为 (Book) -> String
```
* 构造函数：::Regex  双冒号加类名表示引用一个类的构造函数

还可以可以引用一个实例对象的成员函数，此时函数调用的状态会与该对象状态存在关系。例如函数变量引用了 foo::toString，那么执行的时候打印的字符串和foo实例相关。

3. 使用实现函数类型接口的自定义类的实例：
```
//IntTransformer的父接口是一个 (Int) -> Int 函数类型
class IntTransformer: (Int) -> Int {
    override operator fun invoke(x: Int): Int = TODO()
}

val intFunction: (Int) -> Int = IntTransformer()
```

## lambda表达式和匿名函数
lambda 表达式与匿名函数是“函数字面值”，即未声明的函数，也就是一个函数类型的值。

声明一个函数a，接受一个类型为 (String, String) -> Boolean 的函数参数
```
fun a(test: (String, String) -> Boolean) {
    print(test("aa", "aaa"))
}
```
那么要传参的话，可以传递函数：
```
//声明compare函数
fun compare(a: String, b: String): Boolean = a.length < b.length
//引用compare函数值来调用a。引用已声明函数要使用双冒号
a(::compare)
```
或者：
```
//先将匿名函数赋值给compare
val compare = fun(a: String, b: String): Boolean = a.length < b.length
//再将compare传给a函数。可以看到使用函数变量是不用双冒号的
a(compare)
```
如果使用lambda表达式：

```
a({ a, b -> a.length > b.length })
//lambda表达式作为最后一个参数，可以放在圆括号外面，如果是唯一参数，圆括号可省略
//a { a, b -> a.length > b.length }
```

#### lamda表达式语法
lambda表达式的语法关键：
1. 一个lambda表达式必须通过{}来包裹
2. 如果lambda表达式生了参数部分的类型，且返回值类型支持类型推导，那么lambda变量（函数变量）就可以省略函数类型声明
3. 如果lambda变量声明了函数类型，那么lambda表达式的参数部分的类型就可以省略

声明一个类型为 (Int, Int) -> Int 的函数变量sum，使用lambda表达式赋值，完整语法形式如下：
```
val sum: (Int, Int) -> Int = { x: Int, y: Int -> x + y }
```
由于类型可以推断，所以可以简化：
```
val sum = { x: Int, y: Int -> x + y }
```

如果lambda表达式的只有一个参数：
```
fun a(test: (String) -> Boolean) {
    print(test("aa"))
}
//常规传入lambda表达式
a { s -> s.isNotEmpty() }
```
此时可以不用声明唯一参数并忽略->，该参数会隐式声明为`it`：
```
a { it.isNotEmpty() }
```

如果 lambda 表达式的参数未使用，那么可以用下划线取代其名称：
```
map.forEach { _, value -> println("$value!") }
```

注意：
```
fun add(f: (Int) -> Unit) {
    f.invoke(1)
}
val li = { i: Int -> print(i) }
add(li)     //正确
add { li }  //错误，实际是在传给add的lambda中又写了一个为值lambda无用表达式

```

## fun声明函数和lambda的混淆问题
由于使用fun声明函数和lambda表达式都是使用花括号{}，容易产生混淆，总结区别如下：
* fun在没有等号，只有花括号的情况下，是最常见的函数体，如果返回非Unit值，必须带return
* fun带等号，是单表达式函数体，该情况可以省略return

只要使用了等号加花括号的语法，那么构建的就是一个lambda表达式，lambda表达式的参数在花括号内部声明。所以，如果左侧是fun且使用了等号加花括号，那么就是一个lambda表达式函数体，也就是函数的值就是lambda表达式，因为lambda表达式就是函数字面量。

例如函数柯里化的两种写法：
```
//常规函数体
fun sum1(x: Int): (Int) -> Int {
        return { y ->
            x + y
        }
    }

//lambda表达式函数体
fun sum2(x: Int) = { y: Int -> x + y }

val result = sum2(1)
print(result(11))
```

#### 带有接收者的函数字面值
在这样的函数字面值内部，传给调用的接收者对象成为隐式的this
```
//声明函数必须被一个Int变量调用，作用类似于扩展函数
val sum = fun Int.(other: Int): Int = this + other
2.sum(1)//结果为3
```

还可以通过这样的方式，做成dsl语言的形式：
```
class HTML {
    fun head() { …… }
    fun body() { …… }
}

fun html(init: HTML.() -> Unit): HTML {
    val html = HTML()  // 创建接收者对象
    html.init()        // 将该接收者对象传给该 lambda
    return html
}

html {
    head()   
    body()
}

//也可以这样使用，但一般使用lambda才更加灵活
html(HTML::body)
```
html函数的参数类型，上下文为HTML类，所以此init函数将只能被HTML实例调用。调用html函数时，传入的 lambda 表达式就是init函数的字面量，这个lambda的上下文或者说this就是调用init函数的HTML实例。

例如内置扩展函数 apply 等，则结合了扩展函数和这里的带接收者的函数字面值的写法。

## 内联函数
在Kotlin中每声明一个Lambda表达式，就会在字节码中产生一个匿名类。该匿名类包含了一个invoke函数，作为Lambda的调用函数，每次调用时还会创建新对象。所以Lambda表达式存在额外的开销。

Kotlin中的内联函数之所以被设计出来，主要是为了优化Kotlin支持Lambda表达式之后所带来的开销。Java中通过invokedynamic来解决lambda的额外开销，但由于Kotlin要兼容Java6，所以使用内联函数。我们可以用inline关键字来修饰函数，这些函数就成为了内联函数，内联函数本身和传给它的 lambda 表达式都将内联到调用处。它们的函数体在编译期被嵌入每一个被调用的地方，以减少额外生成的匿名类数，以及函数执行的时间开销。

以下情况我们应避免使用内联函数：
1. 由于JVM对普通的函数已经能够根据实际情况智能地判断是否进行内联优化，所以我们并不需要对其使用Kotlin的inline语法，那只会让字节码变得更加复杂。一般对参数为函数类型的函数内联优化。
2. 尽量避免对具有大量函数体的函数进行内联，这样会导致过多的字节码数量；
3. 一旦一个函数被定义为内联函数，便不能获取闭包类的私有成员，除非你把它们声明为internal。

#### noinline：避免参数被内联
如果在一个函数的开头加上inline修饰符，那么它的函数体及Lambda参数都会被内联。但是如果函数接受多个参数，但是只想对部分参数内联，则对该参数使用noinline修饰。
```
fun main() {
    foo({
        println("I am inlined...")
    }, {
        println("I am not inlined...")
    })
}

inline fun foo(block1: () -> Unit, noinline block2: () -> Unit) {//对block2使用了noinline修饰
    println("before block")
    block1()
    block2()
    println("end block")
}
```
然后看到反编译的Java代码：
```
public static void main(String[] var0) {
    Function0 block2$iv = (Function0)null.INSTANCE;
    String var2 = "before block";
    System.out.println(var2);
    String var5 = "I am inlined...";//block1被内联了
    System.out.println(var5);
    block2$iv.invoke();             //block2没有内联
    var2 = "end block";
    System.out.println(var2);
}

public static final void foo(@NotNull Function0 block1, @NotNull Function0 block2) {
    int $i$f$foo = 0;
    Intrinsics.checkParameterIsNotNull(block1, "block1");
    Intrinsics.checkParameterIsNotNull(block2, "block2");
    String var3 = "before block";
    System.out.println(var3);
    block1.invoke();
    block2.invoke();
    var3 = "end block";
    System.out.println(var3);
}
```

#### 非局部返回
局部返回就是函数内的return只会结束所在函数的执行，而不会结束更外层函数。
```
fun main() {
    foo()
}

fun localReturn() {
    return
}

fun foo() {
    println("before local return")
    localReturn()
    println("after local return")
    return
}
//运行结果
before local return
after local return
```
如果换成Lambda的版本：
```
fun main() {
    foo { return }
}

fun foo(returning: () -> Unit) {
    println("before local return")
    returning()
    println("after local return")
    return
}
//Error:(2, 11) Kotlin: 'return' is not allowed here
```
此时编译报错，Lambda中不能存在return，此时可以使用内联函数。将foo函数声明为inline即可正常使用。
```
fun main() {
    foo { return }
}

inline fun foo(returning: () -> Unit) {
    println("before local return")
    returning()
    println("after local return")
    return
}
//运行结果
before local return
```
结果与局部返回不同，Lambda中的return让foo函数也退出了。这就是因为内联函数的原因，直接将Lambda中的return放到了foo函数中。

#### crossinline
由于非局部返回也可能存在危险，所以可以用crossinline修饰参数，拒绝内联局部跳转，此时即使是内联函数，仍不能编译通过Lambda中的return。

#### 具体化参数类型 
Kotlin与Java一样，由于运行时的类型擦除，我们并不能直接获取一个参数的类型。然而，由于内联函数会直接在字节码中生成相应的函数体实现，这种情况下我们反而可以获得参数的具体类型。我们可以用reified修饰符来实现这一效果。
```
fun main() {
    getType<Int>()
}

inline fun <reified T> getType() {
    println(T::class)
}
//运行结果
class kotlin.Int
```
在Android开发中，就可以实现如下效果：
```
inline fun <reified T : Activity> Activity.startActivity() {
    startActivity(Intent(this, T::class))
}
//使用
startActivity<DetailActivity>()
```