## 闭包
闭包 就是匿名函数，默认不可逃逸
#### 逃逸闭包
逃逸闭包：函数返回后才执行，比如异步函数

创建默认不可逃逸闭包的好处： 编译器优化你的代码的性能和能力。如果编译器知道这个闭包是不可逃逸的，它可以关注内存管理的关键细节。

并且可以在不可逃逸闭包里放心的使用self关键字，因为这个闭包总是在函数return之前执行，我们不需要去使用一个弱引用去引用self。

闭包会强引用它捕获的所有对象，比如你在闭包中访问了当前控制器的属性、函数，编译器会要求你在闭包中显示 self 的引用，这样闭包会持有当前对象，容易导致循环引用。

非逃逸闭包不会产生循环引用，它会在函数作用域内释放，编译器可以保证在函数结束时闭包会释放它捕获的所有对象；使用非逃逸闭包的另一个好处是编译器可以应用更多强有力的性能优化，例如，当明确了一个闭包的生命周期的话，就可以省去一些保留（retain）和释放（release）的调用；此外非逃逸闭包它的上下文的内存可以保存在栈上而不是堆上。

#### 自动闭包
可以把参数转换为闭包。它不接受任何实际参数，并且当它被调用时，它会返回内部打包的表达式的值。这个语法的好处在于通过写普通表达式代替显式闭包而使你省略包围函数形式参数的括号。

使用自动闭包前：
```
// customersInLine is ["Alex", "Ewa", "Barry", "Daniella"]
func serve(customer customerProvider: () -> String) {
    print("Now serving \(customerProvider())!")
}
serve(customer: { customersInLine.remove(at: 0) } )
// Prints "Now serving Alex!"
```

使用自动闭包后
```
// customersInLine is ["Ewa", "Barry", "Daniella"]
func serve(customer customerProvider: @autoclosure () -> String) {
    print("Now serving \(customerProvider())!")
}
serve(customer: customersInLine.remove(at: 0))
// Prints "Now serving Ewa!"
```

## 结构体和类
* 结构体和枚举都是值类型，类是引用类型
* Swift 中所有的基本类型——整数，浮点数，布尔量，字符串，数组和字典——都是值类型，并且都以结构体的形式在后台实现
* 引用同一个对象，使用===和!==判断

## 属性
* Swift的属性，区分了存储属性和计算属性，存储属性就是普通的实例属性，而计算属性就是setter和getter功能的属性。对比Kotlin，Swift的计算属性必须有一个直接或间接对应的存储属性，而Kotlin的属性有幕后字段，所以Kotlin的属性既可以是实例属性也可以是计算属性。
* willSet 和 didSet 可以作为属性观察者
* 属性包装

## 方法
结构体是值类型，但可以使用`mutating`修饰的方法来修改自身的值（实际就是返回一个新的值，只是顺便帮我们赋值到原本的变量）。
```
struct Point {
    var x = 0.0, y = 0.0
    mutating func moveBy(x deltaX: Double, y deltaY: Double) {
        x += deltaX
        y += deltaY
    }
}
var somePoint = Point(x: 1.0, y: 1.0)
somePoint.moveBy(x: 2.0, y: 3.0)
print("The point is now at (\(somePoint.x), \(somePoint.y))")
// prints "The point is now at (3.0, 4.0)"
```

## 下标
下标语法，可以自定义`subscript`，类似计算属性拥有setter和getter，并且可以接收参数；使用时通过`[]`来获取返回值。

## 继承
Swift的继承，可以重写方法、属性、属性观察器，并可以用final阻止重写

## 初始化
* 结构体有存储属性，即使没有默认值，也可以不写init函数，因为会自动生成，在构造时就必须传入参数。而类的属性只要没有初始值（可选类型自动初始化为nil），就必须明确声明init函数。
* 子类init和父类init函数参数相同时，需要使用override表示重写。
* Swift的init函数中，初始化当前类的属性，必须在调用super.init之前，这和Java等语言不同。这是为了避免问题：子类重写父类的方法，在重写方法中访问自身的属性，而此方法在父类中会在init中调用，导致子类还没有初始化就访问了该属性。
* 便捷初始化器，使用convenience修饰init函数，强制要求当前init函数调用当前类中的其他init函数，也就是委托给其他init函数。
* Java/Kotlin的类，只要父类没有`无参构造函数`，子类都必须声明自己的构造函数，并调用父类的`有参构造函数`，即使只是简单的包装调用。而Swift在子类没有定义任何init函数时，是可以自动继承父类init函数，而不用强制显式写init函数。
* Java/Kotlin的子类不会自动继承父类的构造函数（没有构造函数时，无参构造函数为自动生成，不算继承），必须显式声明；而Swift的子类，在特定情况下，可以继承父类的init函数。`假设你为你子类引入的任何新的属性都提供了默认值，请遵守以下2个规则（文字描述可能有点抽象，通过实际测试会更容易理解）`：
  1. 如果你的子类没有定义任何指定初始化器，它会自动继承父类所有的指定初始化器。
  2. 如果你的子类提供了所有父类指定初始化器的实现——要么是通过规则1继承来的，要么通过在定义中提供自定义实现的——那么它自动继承所有的父类便捷初始化器。
  > 就算你的子类添加了更多的便捷初始化器，这些规则仍然适用。
* init?表示初始化可以失败，返回nil。
* 在init前添加`required`修饰符来表明所有该类的子类都必须实现该初始化器。
* 可以通过闭包（函数）来初始化属性。

## 反初始化
Swift采用引用计数法来释放内存，所以相对于Java系语言，反初始化的调用时机更可靠，也就更能实际使用。

## 扩展
扩展可以为现有的类、结构体、枚举类型或协议添加新功能。Swift 中的扩展可以：
* 添加计算实例属性和计算类型属性；
* 定义实例方法和类型方法；
* 提供新初始化器；
* 定义下标；
* 定义和使用新内嵌类型；
* 使现有的类型遵循某协议

## 协议
协议相当于Java/Kotlin中的接口，但功能细节更多。

Swift中的协议，可以使用Self，来达到实现协议时，确定Self类型为当前类型，也算是泛型语法的一种。对比Kotlin和Swift：
```
//Kotlin

interface Animal {
    fun isSibling(animal: Animal): Boolean
}

class Dog : Animal {
    override fun isSibling(animal: Animal): Boolean {
        return true
    }
}
```
Kotlin的Dog类，实现isSibling，参数类型只能是Animal。如果要让类型可以是Dog，就要使用泛型。

```
//Swift

protocol Animal {
    func isSibling(_ animal: Self) -> Bool
}

class Dog: Animal {
    func isSibling(_ animal: Dog) -> Bool {
        return true
    }
}
```
Swift通过Self，可以让子类实现方法时，参数类型使用当前类型。所以Self也就是一种泛型语法。

## 泛型
* 泛型可以使用where提供更多约束
* 协议中使用泛型，是使用associatedtype语法
* 泛型协议（使用关联类型，或者Self）不能作为类型来修饰变量、返回值等。因为关联类型不确定，只有被子类实现后才是确定的。

泛型的类型参数由调用者确定，associatedtype关联类型由实现者决定。Swift在协议中不使用尖括号语法而是关联类型来表达泛型的具体原因待探究。

## 不透明类型
泛型协议不能作为类型来修饰变量、返回值等，因为不能确定泛型协议的具体类型（泛型协议的实现者才能确定）。所以增加了some关键字，限制函数返回泛型协议的某一种确定的实现类型，编译器知道类型是确定的，所以可以通过编译。并且使用some后，可知调用该函数返回的类型是确定的一种，也就能确定多次调用该函数返回的是同一类型，可安全使用相互赋值等操作。

## 自动引用计数
* weak 弱引用的类型必须是可空。建议在假定引用可能先于自身释放的情况下使用。
* unowned 无主引用的类型不限制是否可空。建议在可以保证引用比自身生命周期长的情况使用，也就是只要引用被释放，自身就不可能再被使用的情况。
* 闭包如果捕获了当前类的属性（比如使用self），并且当前类的某个属性引用了此闭包，那么也会造成循环强引用。可以使用捕获列表来解决：
```
lazy var someClosure: (Int, String) -> String = {
    [unowned self, weak delegate = self.delegate!] (index: Int, stringToProcess: String) -> String in
    // closure body goes here
}
```