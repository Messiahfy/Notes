## 注意项

### 1. 默认导入
```
java.io.*
java.lang.*
java.math.BigDecimal
java.math.BigInteger
java.net.*
java.util.*
groovy.lang.*
groovy.util.*
```

### 2. Groovy方法重载是运行时确定，Java是编译时确定。

### 3. 依赖管理
Grape是Groovy自带的JAR依赖管理器。使用方式：
```
@Grab(group='org.springframework', module='spring-orm', version='5.2.8.RELEASE')
import org.springframework.jdbc.core.JdbcTemplate

// 或者简写
@Grab('org.springframework:spring-orm:5.2.8.RELEASE')
import org.springframework.jdbc.core.JdbcTemplate
```
默认从maven central仓库查找依赖，如果要设置其他仓库：
```
@GrabResolver(name='restlet', root='http://maven.restlet.org/')
@Grab(group='org.restlet', module='org.restlet', version='1.1.6')
```
更多使用细节请查看：https://groovy-lang.org/grape.html

### 4. 代码风格建议
https://groovy-lang.org/style-guide.html

常用的一些：
* 没有分号
* 方法返回的return关键字可以看情况写或者不写
* 默认是public
* 括号有时可以省略
```
println "Hello" // println("Hello")
method a, b // method(a, b)

list.each( { println it } )
list.each(){ println it }
list.each  { println it }
```
但如果方法不含参数，或者嵌套调用，则不能省略括号
```
def foo(n) { n }
def bar() { 1 }

println foo 1 // won't work
def m = bar   // won't work
```

* 支持类似Kotlin的getter/setter
* 有类似Kotlin的作用域函数with、tap
* equals和`==`：groovy的is()函数就是Java的==，groovy的==就是Java的 equals
* 支持import别名
* 支持`?.`可空调用和`?:`

## 1. 基本语法

### 1.1 引号
groovy的引号跟在点号后面有一些有趣的效果：
`person.name`和`person."name"`和`person.'name'`的效果一样
而且可以使在Java中非法的情况在groovy中被允许
```
def map = [:]
map."an identifier with a space and double quotes" = "ALLOWED"
map.'with-dash-signs-and-single-quotes' = "ALLOWED"

assert map."an identifier with a space and double quotes" == "ALLOWED"
assert map.'with-dash-signs-and-single-quotes' == "ALLOWED"
```

### 1.2 字符串
可以实例化`java.lang.String`和`groovy.lang.GString`类型的字符串。
#### 1.2.1 单引号字符串
单引号字符串是普通的`java.lang.String`，不支持插值。
```
`abc`
```

#### 1.2.2 字符串拼接

```
assert 'ab' == 'a' + 'b'
```

#### 1.2.3 三重单引号字符串
```
'''a triple single quoted string'''
```
三重单引号字符串是普通的`java.lang.String`，不支持插值。
三重单引号字符串可以是多行。 您可以跨越行边界跨越字符串的内容，而无需将字符串拆分为多个部分，而不需要连接或换行转义字符：
```
def aMultilineString = '''line one
line two
line three'''
```

#### 1.2.4 双引号字符串
没有插值的双引号字符串为`java.lang.String`实例，有插值的双引号字符串为`groovy.lang.GString`实例。

Groovy表达式可以在双引号中字符串插值， 插值是在对字符串求值时将字符串中的占位符替换为其值的行为。 占位符表达式由`${}`包围，或者以`$`表示前缀。

调用GString的`toString()`方法可以将占位符内的表达式值计算为其字符串表示形式。
```
def name = 'Guillaume' // a plain string
def greeting = "Hello ${name}"

assert greeting.toString() == 'Hello Guillaume'
```
任何Groovy表达式都是有效的，正如我们在本例中可以看到的算术表达式：
```
def sum = "The sum of 2 and 3 equals ${2 + 3}"
assert sum.toString() == 'The sum of 2 and 3 equals 5'
```
使用`$`作为点式表达式（访问属性）的前缀
```
def person = [name: 'Guillaume', age: 36]
assert "$person.name is $person.age years old" == 'Guillaume is 36 years old'
```
### 1.3 Lists
&emsp;&emsp;`Groovy`使用逗号分隔的值列表（用方括号括起来）来表示列表。 `Groovy`列表是普通的`JDK java.util.List`，因为`Groovy`没有定义自己的集合类。 默认情况下，定义列表文字时使用的具体列表实现是`java.util.ArrayList`，除非您决定另行指定，我们将在后面看到。
```
def numbers = [1, 2, 3]         

assert numbers instanceof List  
assert numbers.size() == 3   
```
设置`List`为其他类型，使用`as`，或者类型声明
```
def arrayList = [1, 2, 3]
assert arrayList instanceof java.util.ArrayList

def linkedList = [2, 3, 4] as LinkedList    
assert linkedList instanceof java.util.LinkedList

LinkedList otherLinked = [3, 4, 5]          
assert otherLinked instanceof java.util.LinkedList
```

可以通过`[]`来访问读写元素，使用`<<`来添加元素
```
def letters = ['a', 'b', 'c', 'd']

assert letters[0] == 'a'     
assert letters[1] == 'b'

assert letters[-1] == 'd'    
assert letters[-2] == 'c'

letters[2] = 'C'             
assert letters[2] == 'C'

letters << 'e'               
assert letters[ 4] == 'e'
assert letters[-1] == 'e'

assert letters[1, 3] == ['b', 'd']         
assert letters[2..4] == ['C', 'd', 'e']
```
### 1.4 Arrays
Groovy对于数组重用了列表的表示法，但是为了制作这样的文字数组，你需要通过强制或类型声明来明确地定义数组的类型。
```
String[] arrStr = ['Ananas', 'Banana', 'Kiwi']  

assert arrStr instanceof String[]    
assert !(arrStr instanceof List)

def numArr = [1, 2, 3] as int[]      

assert numArr instanceof int[]       
assert numArr.size() == 3
```

### 1.5 Maps
```
def colors = [red: '#FF0000', green: '#00FF00', blue: '#0000FF']   

assert colors['red'] == '#FF0000'    
assert colors.green  == '#00FF00'    

colors['pink'] = '#FF00FF'           
colors.yellow  = '#FFFF00'           

assert colors.pink == '#FF00FF'
assert colors['yellow'] == '#FFFF00'

assert colors instanceof java.util.LinkedHashMap
```
Groovy创建的map类型实质是`java.util.LinkedHashMap`

## 2.对象操作符
### 2.1 安全导航操作符（Safe Navigation operator）
被用于避免`NullPointerException`，
```
def name = person?.name              //假设person为null        
assert name == null
```
返回`null`结果，而不是抛出错误，使用`null-safe`运算符可防止出现`NullPointerException`。

### 2.2 直接公有域访问操作符
使用`user.name`默认调用`getter`，使用`@`操作符则强制使用域本身而不是`getter`（有时`getter`返回的值经过自定义，可能和域本身有差别）
```
assert user.@name == 'Bob' 
```

### 2.3 方法指针操作符
方法指针运算符（`.＆`）调用用于存储对变量中方法的引用，以便稍后调用它：
```
def str = 'example of method reference'            
def fun = str.&toUpperCase                         
def upper = fun()                                  
assert upper == str.toUpperCase()
```

使用方法指针有许多优点。 首先，这种方法指针的类型是groovy.lang.Closure，因此它可以在任何地方使用闭包。 特别是，它适合转换现有方法以满足策略模式的需要：
```
def transform(List elements, Closure action) {                    
    def result = []
    elements.each {
        result << action(it)
    }
    result
}
String describe(Person p) {                                       
    "$p.name is $p.age"
}
def action = this.&describe                                       
def list = [
    new Person(name: 'Bob',   age: 42),
    new Person(name: 'Julia', age: 35)]                           
assert transform(list, action) == ['Bob is 42', 'Julia is 35']
```

## 3.其他操作符
### 3.1 Spread operator
用于调用聚合对象的所有项目上的操作。 它相当于对每个项目调用操作并将结果收集到列表中：
```
class Car {
    String make
    String model
}
def cars = [
       new Car(make: 'Peugeot', model: '508'),
       new Car(make: 'Renault', model: 'Clio')]       
def makes = cars*.make                                
assert makes == ['Peugeot', 'Renault']
```

### 3.2 Call operator
调用操作符`()`用于隐式调用名为`call`的方法。 对于定义调用方法的任何对象，可以省略.call部分并使用调用运算符：
```
class MyCallable {
    int call(int x) {           
        2*x
    }
}

def mc = new MyCallable()
assert mc.call(2) =
```