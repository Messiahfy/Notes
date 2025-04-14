## 1.概述
&emsp;&emsp;`Groovy`中的闭包是一个开放的，匿名的代码块，可以接受参数，返回值并分配给变量。闭包可以引用在其包围范围内声明的变量。与闭包的正式定义相反，Groovy语言中的Closure也可以包含在其包围范围之外定义的变量。

## 2.语法
### 2.1定义一个闭包
闭包定义遵循以下语法：
> { [closureParameters -> ] statements }

其中`[closureParameters->]`是一个以逗号分隔的可选参数列表，而`statements`是0或多个Groovy语句。 参数看起来类似于方法参数列表，这些参数可以是类型化的或非类型化的。

指定参数列表时， `->`字符是必需的，用于将参数与闭包实体分开。

合法闭包定义的一些示例：
```
{ item++ }     //一个引用 名字是item的变量 的闭包                                     

{ -> item++ }      //通过添加 -> 来明确将参数和语句代码分发
```
使用隐式参数`it`的闭包
```
{ println it }
```
一个替代版本，`it`是显式参数
```
{ it -> println it }       
```
```
{ name -> println name } //在这种情况下，通常最好为参数使用显式名称

{ String x, int y ->                                
    println "hey ${x} the value is ${y}"    //两个类型化参数
}

{ reader ->                                         
    def line = reader.readLine()   //一个闭包可以包含多个语句
    line.trim()
}
```

### 2.2闭包作为对象
&emsp;&emsp;一个闭包是`groovy.lang.Closure`类的实例，它可以像任何其他变量一样分配给变量或字段，尽管它是一个代码块：
```
def listener = { e -> println "Clicked on $e.source" }      
assert listener instanceof Closure
Closure callback = { println 'Done!' }     
                 
Closure<Boolean> isTextFile = {
    File it -> it.name.endsWith('.txt')                     
}
```
最后的闭包：可选的，您可以使用`groovy.lang.Closure`的泛型类型指定闭包的返回类型

### 2.3 调用闭包
&emsp;&emsp;作为匿名代码块的闭包可以像任何其他方法一样被调用。 如果你定义一个不带参数的闭包，如下：
```
def code = { 123 }
```
然后闭包内的代码只会在你调用闭包时执行，这可以通过使用变量来完成，就好像它是一个常规方法：
```
assert code() == 123
```
或者，您可以显式并使用调用方法：
```
assert code.call() == 123
```
如果闭包接收参数，原则是相同的：
```
def isOdd = { int i -> i%2 != 0 }                           
assert isOdd(3) == true                                     
assert isOdd.call(2) == false                               

def isEven = { it%2 == 0 }                                  
assert isEven(3) == false                                   
assert isEven.call(2) == true 
```
`isOdd`和`isEven`两个闭包，后者使用隐式参数`it`。

与方法不同，闭包在调用时始终返回一个值。

## 3.参数
### 3.1 普通参数
闭包的参数遵循与常规方法的参数相同的原则：
* 可选类型
* 一个名字
* 可选的默认值

参数用逗号分隔：
```
def closureWithOneArg = { str -> str.toUpperCase() }
assert closureWithOneArg('groovy') == 'GROOVY'

def closureWithOneArgAndExplicitType = { String str -> str.toUpperCase() }
assert closureWithOneArgAndExplicitType('groovy') == 'GROOVY'

def closureWithTwoArgs = { a,b -> a+b }
assert closureWithTwoArgs(1,2) == 3

def closureWithTwoArgsAndExplicitTypes = { int a, int b -> a+b }
assert closureWithTwoArgsAndExplicitTypes(1,2) == 3

def closureWithTwoArgsAndOptionalTypes = { a, int b -> a+b }
assert closureWithTwoArgsAndOptionalTypes(1,2) == 3

def closureWithTwoArgAndDefaultValue = { int a, int b=2 -> a+b }
assert closureWithTwoArgAndDefaultValue(1) == 3
```

### 3.2 隐式参数
&emsp;&emsp;当闭包没有显式定义参数列表（使用 `->`）时，则闭包总是会定义一个名为`it`的隐式参数。 这意味着这段代码：
```
def greeting = { "Hello, $it!" }
assert greeting('Patrick') == 'Hello, Patrick!'
```
和下面的代码是严格相等的
```
def greeting = { it -> "Hello, $it!" }
assert greeting('Patrick') == 'Hello, Patrick!'
```
### 3.3 可变参数
```
def concat1 = { String... args -> args.join('') }           
assert concat1('abc','def') == 'abcdef'                     
def concat2 = { String[] args -> args.join('') }            
assert concat2('abc', 'def') == 'abcdef'

def multiConcat = { int n, String... args ->                
    args.join('')*n
}
assert multiConcat(2, 'abc','def') == 'abcdefabcdef'
```
## 4.委托策略（Delegation strategy）
### 4.1 Groovy闭包对比lambda表达式
&emsp;&emsp;`Groovy`将闭包定义为`Closure`类的实例。 它使它与`Java 8`中的`lambda`表达式截然不同。委托是`Groovy`闭包中的一个关键概念，它在`lambda`中没有等价物。 能够更改委托或更改闭包的委派策略使得在Groovy中设计漂亮的域特定语言（DSL）成为可能。

### 4.2 Owner, delegate and this
&emsp;&emsp;要理解委托的概念，首先必须在闭包中解释这个含义。 闭包实际上定义了3个不同的东西：
* `this`对应于定义了闭包的类（闭包在此类中）

* `owner`对应于定义闭包的封闭对象（闭包在它之内），要么是类要么是一个闭包。

* `delegate`默认是和`owner`一致，或者自定义`delegate`指向

#### 4.2.1 this的意义
在闭包中，调用`getThisObject`将返回定义闭包的包围类。 它相当于使用一个明确的`this`：
```
class Enclosing {
    void run() {
        def whatIsThisObject = { getThisObject() }          
        assert whatIsThisObject() == this                   
        def whatIsThis = { this }                           
        assert whatIsThis() == this                         
    }
}
class EnclosedInInnerClass {
    class Inner {
        Closure cl = { this }                               
    }
    void run() {
        def inner = new Inner()
        assert inner.cl() == inner                          
    }
}
class NestedClosures {
    void run() {
        def nestedClosures = {
            def cl = { this }                               
            cl()
        }
        assert nestedClosures() == this                     
    }
}
```
> `this`只会对应定义闭包的最近包围类，不会是外部闭包

#### 4.2.2 闭包的Owner
`Owner`和`this`很像，但有一个小区别：`Owner`对应最近的包围对象，可以是类或者闭包
```
class Enclosing {
    void run() {
        def whatIsOwnerMethod = { getOwner() }               
        assert whatIsOwnerMethod() == this                   
        def whatIsOwner = { owner }                          
        assert whatIsOwner() == this                         
    }
}
class EnclosedInInnerClass {
    class Inner {
        Closure cl = { owner }                               
    }
    void run() {
        def inner = new Inner()
        assert inner.cl() == inner                           
    }
}
class NestedClosures {
    void run() {
        def nestedClosures = {
            def cl = { owner }                               
            cl()
        }
        assert nestedClosures() == nestedClosures            
    }
}
```
owner可以是闭包，这和this不一样

#### 4.2.3 闭包的委托
可以使用`delegate`属性或调用`getDelegate`、`setDelegate`方法来访问闭包的委托。 它是`Groovy`中构建特定领域语言的强大概念。 `closure-this`和`closure-owner`引用了闭包的词法范围，而委托是一个闭包使用的用户定义的对象。 默认情况下，委托设置为`Owner`：

默认情况：
```
class Enclosing {
    void run() {
        def cl = { getDelegate() }                          
        def cl2 = { delegate }                              
        assert cl() == cl2()                                
        assert cl() == this                                 
        def enclosed = {
            { -> delegate }.call()                          
        }
        assert enclosed() == enclosed                       
    }
}
```

闭包的委托可以更改为任何对象。让我们通过创建两个不是彼此的子类但都定义名为`name`的属性来说明这一点：
```
class Person {
    String name
}
class Thing {
    String name
}

def p = new Person(name: 'Norman')
def t = new Thing(name: 'Teapot')
```
然后定义一个闭包，它在委托上获取`name`属性：
```
def upperCasedName = { delegate.name.toUpperCase() }
```
然后通过更改闭包的委托，您可以看到目标对象将更改：
```
upperCasedName.delegate = p
assert upperCasedName() == 'NORMAN'
upperCasedName.delegate = t
assert upperCasedName() == 'TEAPOT'
```

#### 4.2.4 委托策略
每当在闭包中访问属性而不显式设置接收者对象时，就会涉及委托策略：
```
class Person {
    String name
}
def p = new Person(name:'Igor')
def cl = { name.toUpperCase() }                 
cl.delegate = p                                 
assert cl() == 'IGOR'
```
此代码工作的原因是`name`属性将在委托对象上透明地解析！ 这是解决闭包内属性或方法调用的一种非常强大的方法。 无需设置显式委托。 接收者（receiver）：将进行调用，因为闭包的默认委托策略就是这样。 闭包实际上定义了多种解析策略，您可以选择：

* `Closure.OWNER_FIRST`：默认策略。 如果`owner`上存在属性/方法，则将在所有者上调用它。 如果没有，则使用`delegete`。
* `Closure.DELEGATE_FIRST`：颠倒逻辑：首先使用`delegete`，然后使用`owner`
* `Closure.OWNER_ONLY`：仅在`owner`上寻找属性/方法，忽略`delegete`
* `Closure.DELEGATE_ONLY`：仅在`delegete`上寻找属性/方法，忽略`owner`
* `Closure.TO_SELF`：不会在`delegete`和`owner`上寻找，仅在闭包类本身寻找属性/方法，

调用`closure.resolveStrategy = Closure.DELEGATE_ONLY`或者`closure.setResolveStrategy(Closure.DELEGATE_ONLY)`可以修改策略。
[闭包](https://www.jianshu.com/p/6dc2074480b8)·

## 5. 其他
Groovy的闭包同样拥有柯里化、函数式编程之类的作用。

https://groovy-lang.org/dsls.html

## 链式命令
Groovy支持在顶层语句中的方法调用参数的括号，例如`a b c d`实际上相当于`a(b).c(d)`
```
// 等同于: turn(left).then(right)
turn left then right

// 命名参数
// 等同于: check(that: margarita).tastes(good)
check that: margarita tastes good

// 闭包作为参数
// 等同于: given({}).when({}).then({})
given { } when { } then { }
```

无参方法需要使用括号：
```
// 等同于: select(all).unique().from(names)
select all unique() from names
```

奇数个元素，最终以属性访问结束：
```
// 等同于:  take(3).cookies
// 或者: take(3).getCookies()
take 3 cookies
```

创建一个dsl（使用Map和闭包）：
```
show = { println it }
square_root = { Math.sqrt(it) }

def please(action) {
  [the: { what ->
    [of: { n -> action(what(n)) }]
  }]
}

// 等同于: please(show).the(square_root).of(100)
please show the square_root of 100
// ==> 10.0
```

## 操作符重载
groovy的操作符都对应调用一个函数，比如+对于plus

## Script 基类
脚本文件会编译为继承groovy.lang.Script 的类，包含run方法，编译后脚本的内容会放到run方法中，脚本中的方法将放到实现子类中，

## @DelegatesTo
```
email {
    from 'dsl-guru@mycompany.com'
    to 'john.doe@waitaminute.com'
    subject 'The pope has resigned!'
    body {
        p 'Really, the pope has resigned!'
    }
}
```

```
def email(Closure cl) {
    def email = new EmailSpec()
    // 将email设置为delegate
    def code = cl.rehydrate(email, this, this)
    // 并且设置闭包策略为仅使用delegate
    code.resolveStrategy = Closure.DELEGATE_ONLY
    code()
}
```

```
class EmailSpec {
    void from(String from) { println "From: $from"}
    void to(String... to) { println "To: $to"}
    void subject(String subject) { println "Subject: $subject"}
    void body(Closure body) {
        def bodySpec = new BodySpec()
        def code = body.rehydrate(bodySpec, this, this)
        code.resolveStrategy = Closure.DELEGATE_ONLY
        code()
    }
}
```

问题在于闭包中能调用的方法没有任何信息，只能看文档，另外，不能帮助IDE代码提示。所以Groovy引入了@DelegatesTo注解，
```
def email(@DelegatesTo(strategy=Closure.DELEGATE_ONLY, value=EmailSpec) Closure cl) {
    def email = new EmailSpec()
    def code = cl.rehydrate(email, this, this)
    code.resolveStrategy = Closure.DELEGATE_ONLY
    code()
}
```

