## 基本

### 常量和变量
```
let a = 10 // 常量
var b = 20 // 变量
```

### 类型别名
```
typealias AudioSample = UInt16
var maxAmplitudeFound = AudioSample.min
```

### 元组
```
let http404Error = (404, "Not Found")
let (statusCode, statusMessage) = http404Error

// 可以命名
let http200Status = (statusCode: 200, description: "OK")
http200Status.statusCode
http200Status.description
```

### 可空
```
let name: String? = nil
let greeting = name ?? "friend"


var b: Int? = 1
let a = b! // 强制非空
```

### 区间运算符
```
// 闭区间运算符
for index in 1...5 {
    print("\(index) times 5 is \(index * 5)")
}

// 半开闭区间运算符
for i in 0..<count {
    print("Person \(i + 1) is called \(names[i])")
}

// 单侧区间
for name in names[2...] {
    print(name)
}
for name in names[...2] {
    print(name)
}
```

### 字符串

#### 字符串插值
```
// 圆括号包裹，然后使用反斜杠前缀
let multiplier = 3
let message = "\(multiplier) times 2.5 is \(Double(multiplier) * 2.5)"
```

#### 字符串是值类型
Swift 的 String类型是一种值类型。如果你创建了一个新的 String值， String值在传递给方法或者函数的时候会被复制过去，还有赋值给常量或者变量的时候也是一样。每一次赋值和传递，现存的 String值都会被复制一次，传递走的是拷贝而不是原本。另一方面，Swift 编译器优化了字符串使用的资源，实际上拷贝只会在确实需要的时候才进行。

#### 子字符串
？？？？

## 集合
### 数组
```
// 创建数组
var someInts = [Int]()
// 或者 var someInts = Array<Int>()
someInts.append(3)

// 使用默认值创建数组
var threeDoubles = Array(repeating: 0.0, count: 3)

// 加运算符（+）可以拼接两个数组
var anotherThreeDoubles = Array(repeating: 2.5, count: 3)
var sixDoubles = threeDoubles + anotherThreeDoubles

// 使用数组字面量创建数组（可以省略类型）
var shoppingList: [String] = ["Eggs", "Milk"]
```

### Set
```
var letters = Set<Character>()
letters.insert("a")

// 使用数组字面量创建Set
var favoriteGenres: Set<String> = ["Rock", "Classical", "Hip hop"]
```

### 字典
```
var namesOfIntegers = [Int: String]()
namesOfIntegers[16] = "sixteen"

// 用字典字面量创建字典
var airports: [String: String] = ["YYZ": "Toronto Pearson", "DUB": "Dublin"]
```

## 控制流
### 常规控制流
* for in
* while
* repeat-while，类似其他语言的do-while


### fallthrough
Swift 中的 Switch 语句不会从每个情况的末尾贯穿到下一个情况中。相反，整个 switch 语句会在第一个匹配到的情况执行完毕之后就直接结束执行。比较而言，C 你在每一个 switch 情况末尾插入显式的 break 语句来阻止贯穿。避免默认贯穿意味着 Swift 的 switch 语句比 C 更加清晰和可预料，并且因此它们避免了意外执行多个 switch 情况。

如果你确实需要 C 风格的贯穿行为，你可以选择在每个情况末尾使用 fallthrough 关键字。

### guard
guard 语句，类似于 if 语句，但如果guard的条件不满足，会立即执行else分支，并退出当前作用域。在else分支中必须使用return、break、continue或throw等控制流语句。
```
guard 条件 else {
    // 条件不满足时执行的代码
    // 必须使用控制流语句（如return、break等）
    return
}
// 条件满足时继续执行的代码
```

### defer
defer 语句，用于在当前作用域结束时执行一段代码，if、for等都会创建独立的作用域。并且无论程序如何退出该作用域，defer 语句中的代码都会被执行。（除非程序崩溃或者强制终止）
```
var score = 1
if score < 10 {
    defer {
        print(score)
    }
    score += 5
}
// Prints "6"
```

### 检查API的可用性
```
if #available(iOS 10, macOS 10.12, *) {
    // Use iOS 10 APIs on iOS, and use macOS 10.12 APIs on macOS
} else {
    // Fall back to earlier iOS and macOS APIs
}


// 反向检测
if #unavailable(iOS 10) {
    // Fallback code
}
```

## 函数
```
func greet(person: String) -> String {
    let greeting = "Hello, " + person + "!"
    return greeting
}

// Swift 默认就是命名参数，调用时需要指定参数名
print(greet(person: "Anna"))
```

### 多返回值的函数
```
// 也就是返回元组
func minMax(array: [Int]) -> (min: Int, max: Int) {
    var currentMin = array[0]
    var currentMax = array[0]
    for value in array[1..<array.count] {
        if value < currentMin {
            currentMin = value
        } else if value > currentMax {
            currentMax = value
        }
    }
    return (currentMin, currentMax)
}
```

### 参数标签
for是一个外部参数标签，叫任何名称都可以，外部参数标签可以让函数调用更清晰，类似于自然语言。
```
func greeting(for person: String) -> String {
    "Hello, " + person + "!"
}

print(greeting(for: "Dave"))


// 格式如下：
func someFunction(argumentLabel parameterName: Int) {
    // In the function body, parameterName refers to the argument value
    // for that parameter.
}

// 再来个例子：
func greet(person: String, from hometown: String) -> String {
    return "Hello \(person)!  Glad you could visit from \(hometown)."
}
print(greet(person: "Bill", from: "Cupertino"))
```

### 省略参数标签
```

func someFunction(_ firstParameterName: Int, secondParameterName: Int) {
    // In the function body, firstParameterName and secondParameterName
    // refer to the argument values for the first and second parameters.
}
someFunction(1, secondParameterName: 2)
```

### 默认形式参数值
```
func someFunction(parameterWithDefault: Int = 12) {
    // In the function body, if no arguments are passed to the function
    // call, the value of parameterWithDefault is 12.
}
someFunction(parameterWithDefault: 6) // parameterWithDefault is 6
someFunction() // parameterWithDefault is 12
```

### 可变形式参数
```
func arithmeticMean(_ numbers: Double...) -> Double {
    var total: Double = 0
    for number in numbers {
        total += number
    }
    return total / Double(numbers.count)
}
arithmeticMean(1, 2, 3, 4, 5)
```

### 输入输出形式参数
可变形式参数只能在函数的内部做改变。如果你想函数能够修改一个形式参数的值，而且你想这些改变在函数结束之后依然生效，那么就需要将形式参数定义为输入输出形式参数。
```
func swapTwoInts(_ a: inout Int, _ b: inout Int) {
    let temporaryA = a
    a = b
    b = temporaryA
}

var someInt = 3
var anotherInt = 107
swapTwoInts(&someInt, &anotherInt)
print("someInt is now \(someInt), and anotherInt is now \(anotherInt)")
// prints "someInt is now 107, and anotherInt is now 3"
```
**也就是传递引用**

### 函数类型
```
// 函数也是一种类型
var mathFunction: (Int, Int) -> Int = addTwoInts

// 函数类型作为参数
func printMathResult(_ mathFunction: (Int, Int) -> Int, _ a: Int, _ b: Int) {
    print("Result: \(mathFunction(a, b))")
}

// 函数作为返回值
func chooseStepFunction(backwards: Bool) -> (Int) -> Int {
    return backwards ? stepBackward : stepForward
}
```

### 局部函数
```
func chooseStepFunction(backward: Bool) -> (Int) -> Int {
    func stepForward(input: Int) -> Int { return input + 1 }
    func stepBackward(input: Int) -> Int { return input - 1 }
    return backward ? stepBackward : stepForward
}
```

## 闭包
全局和内嵌函数，实际上是特殊的闭包。闭包符合如下三种形式中的一种：

* 全局函数是一个有名字但不会捕获任何值的闭包；
* 内嵌函数是一个有名字且能从其上层函数捕获值的闭包；
* 闭包表达式是一个轻量级语法所写的可以捕获其上下文中常量或变量值的没有名字的闭包。（匿名函数）

```
// 闭包表达式语法
  { (parameters) -> (return type) in
    statements
  }

// 例子：
reversedNames = names.sorted(by: { (s1: String, s2: String) -> Bool in
    return s1 > s2
})

// 类型推断：
reversedNames = names.sorted(by: { s1, s2 in return s1 > s2 } )

// 单表达式闭包隐式返回，省略 return
reversedNames = names.sorted(by: { s1, s2 in s1 > s2 } )

// 简写实际参数名
reversedNames = names.sorted(by: { $0 > $1 } )

// 使用运算符函数的最简实现
reversedNames = names.sorted(by: >)
```

如果函数接收多个闭包，你可省略第一个尾随闭包的实际参数标签，但要给后续的尾随闭包写标签。比如说，下面的函数给照片墙加载图片：
```
func loadPicture(from server: Server, completion: (Picture) -> Void, onFailure: () -> Void) {
    if let picture = download("photo.jpg", from: server) {
        completion(picture)
    } else {
        onFailure()
    }
}

loadPicture(from: someServer) { picture in
    someView.currentPicture = picture
} onFailure: {
    print("Couldn't download the next picture.")
}
```


### 尾随闭包
```
func someFunctionThatTakesAClosure(closure:() -> Void){
    //function body goes here
}

someFunctionThatTakesAClosure({
     //closure's body goes here
})
  
//here's how you call this function with a trailing closure instead
      
someFunctionThatTakesAClosure() {
     // trailing closure's body goes here
}

// 例子：
reversedNames = names.sorted() { $0 > $1 }
```

如果闭包表达式作为函数的唯一实际参数传入，而你又使用了尾随闭包的语法，那你就不需要在函数名后边写圆括号了：
```
reversedNames = names.sorted { $0 > $1 }
```

如果一个函数接受多个闭包，则省略第一个尾随闭包的参数标签，并为其余尾随闭包添加标签：
```
func loadPicture(from server: Server, completion: (Picture) -> Void, onFailure: () -> Void) {
    if let picture = download("photo.jpg", from: server) {
        completion(picture)
    } else {
        onFailure()
    }
}


loadPicture(from: someServer) { picture in
    someView.currentPicture = picture
} onFailure: {
    print("Couldn't download the next picture.")
}
```

### 捕获值
闭包可以从定义它的上下文中捕获常量和变量
```
func makeIncrementer(forIncrement amount: Int) -> () -> Int {
    var runningTotal = 0
    func incrementer() -> Int {
        runningTotal += amount
        return runningTotal
    }
    return incrementer
}

let incrementByTen = makeIncrementer(forIncrement: 10)

incrementByTen()
//return a value of 10
incrementByTen()
//return a value of 20
incrementByTen()
//return a value of 30

// 再创建一个 incrementBySeven ，它和 incrementByTen 是完全独立的，不会共享捕获的值
let incrementBySeven = makeIncrementer(forIncrement: 7)
incrementBySeven()
// returns a value of 7
```
runningTotal 被 incrementer() 闭包捕获

**无论值类型还是引用类型，默认都是捕获引用，并且会增加强引用计数**：
```
var counter = 0
let closure = {
    print("Counter in closure: \(counter)")
}
counter = 10
closure() // 打印：Counter in closure: 10
```

**但如果使用了捕获列表，则值类型被捕获的是拷贝**（引用类型不受影响）：
```
var counter = 0
let closure = { [counter] in
    // 即使counter是变量，但因为使用捕获列表，捕获的是拷贝，所以闭包内无法修改它
    print("Counter in closure: \(counter)")
}
counter = 10
closure() // 打印：Counter in closure: 0
```

### 闭包是引用类型
函数和闭包都是引用类型，所以赋值后，两个变量/常量都会指向同一个闭包：
```
let alsoIncrementByTen = incrementByTen
alsoIncrementByTen() // 所以再次增加10，会从30变成40
//return a value of 40
```

### 逃逸闭包
逃逸闭包：函数返回后才执行，换句话说，闭包的生命周期超出了函数的执行范围，逃逸闭包通常用于异步操作或延迟执行的情况。

Swift的闭包默认是不可逃逸的，如果编译器知道这个闭包是不可逃逸的，这样利于编译器的内存管理。**如果闭包可能逃逸，必须使用`@escaping`关键字显式声明。**
```
var completionHandlers: [() -> Void] = []
func someFunctionWithEscapingClosure(completionHandler: @escaping () -> Void) {
    completionHandlers.append(completionHandler)
}
```

**在类中使用时，逃逸闭包必须显式地引用 `self`，这样可以更清楚地提示开发者，并且提醒确认这里有没有引用循环**：
```
func someFunctionWithNonescapingClosure(closure: () -> Void) {
    closure()
}
 
class SomeClass {
    var x = 10
    func doSomething() {
        someFunctionWithEscapingClosure { self.x = 100 }
        someFunctionWithNonescapingClosure { x = 200 }

        // 也可以把 self 放到闭包捕获列表，这样闭包内 in 之后的代码就可以省略 self
        someFunctionWithEscapingClosure { [self] in x = 100 }
    }
}
```
可以在不可逃逸闭包里放心的使用self关键字，因为这个闭包总是在函数return之前执行，我们不需要去使用一个弱引用去引用self。

非逃逸闭包不会产生循环引用，它会在函数作用域内释放，编译器可以保证在函数结束时闭包会释放它捕获的所有对象；使用非逃逸闭包的另一个好处是编译器可以应用更多强有力的性能优化，例如，当明确了一个闭包的生命周期的话，就可以省去一些保留（retain）和释放（release）的调用；此外非逃逸闭包它的上下文的内存可以保存在栈上而不是堆上。

**当 self 是结构体或者枚举的实例时，逃逸闭包不能捕获可修改的 self 引用，因为结构体和枚举是值类型，它们的实例在传递时会被复制，而不是像类那样传递引用。并且即使是不可逃逸的闭包，要引用self，也需要在函数使用`mutating`关键字**：
```
struct SomeStruct {
    var x = 10
    mutating func doSomething() {
        someFunctionWithNonescapingClosure { x = 200 }  // Ok
        someFunctionWithEscapingClosure { x = 100 }     // Error
    }
}
```

`mutating`关键字用于修饰结构体（struct）和枚举（enum）中的方法，表示该方法可以修改实例的属性，或者直接重新赋值self。由于结构体和枚举是值类型，默认情况下它们的方法不能修改自身的属性，因此需要使用mutating来明确表示该方法会修改实例的状态。**Swift 设计 mutating 关键字而不是直接让结构体字段可变，主要源于对值类型安全性和明确语义的考量。**

### 自动闭包
`@autoclosure` 可以把参数自动转换为闭包。但它不能有参数，并且当它被调用时，它会返回内部打包的表达式的值。这个语法的好处在于通过写普通表达式代替显式闭包而使你省略包围函数形式参数的括号。

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
闭包内部的代码直到你调用它的时候才会运行。`@autoclosure`可以和`@escaping`同时使用。

## 枚举
**枚举是值类型**

```
enum CompassPoint {
    case north
    case south
    case east
    case west
}

// 或者
enum Planet {
    case mercury, venus, earth, mars, jupiter, saturn, uranus, neptune
}
```
**Swift的枚举，并不会分配一个默认的整数值，比如0、1、2**

### 枚举值简写
```
var directionToHead = CompassPoint.west
// 可以推断的情况下，可以简写枚举
directionToHead = .east
```

### 遍历枚举
要让枚举可以遍历，可以添加 `CaseIterable`：

```
enum Beverage: CaseIterable {
    case coffee, tea, juice
}
let numberOfChoices = Beverage.allCases.count
```

### 关联值
Swift的枚举可以存储任意给定类型的关联值，并且不同枚举成员关联值的类型可以不同：
```
enum Barcode {
    case upc(Int, Int, Int, Int)
    case qrCode(String)
}

var productBarcode = Barcode.upc(8, 85909, 51226, 3)

switch productBarcode {
case .upc(let numberSystem, let manufacturer, let product, let check):
    print("UPC: \(numberSystem), \(manufacturer), \(product), \(check).")
case .qrCode(let productCode):
    print("QR code: \(productCode).")
}
```
> 在Kotlin中，枚举无法这样使用，需要利用sealed class。


如果对于一个枚举成员的所有的相关值都被提取为常量，或如果都被提取为变量，为了简洁，你可以用一个单独的 var 或 let 在成员名称前标注
```
switch productBarcode {
case let .upc(numberSystem, manufacturer, product, check):
    print("UPC : \(numberSystem), \(manufacturer), \(product), \(check).")
case let .qrCode(productCode):
    print("QR code: \(productCode).")
}
```

### 原始值
枚举原始值被定义为类型 Character：
```
enum ASCIIControlCharacter: Character {
    case tab = "\t"
    case lineFeed = "\n"
    case carriageReturn = "\r"
}
```

隐式指定原始值，venus的隐式原始值是 2，依次类推：
```
enum Planet: Int {
    case mercury = 1, venus, earth, mars, jupiter, saturn, uranus, neptune
}
```

south有一个隐式原始值 "south" ，以此类推：
```
enum CompassPoint: String {
    case north, south, east, west
}

CompassPoint.north.rawValue // "north"
```
> 字符串类型、数值类型可以作为隐式的原始值，如果没有分配任何一个显式的原始值，则会根据设置的类型来设置枚举成员的值，比如字符串类型则会使用枚举成员的名称，Int类型则从0开始自增

可以根据原始值来获取枚举成员，但不是所有可能的 Int值都会对应一个行星，所以类型为可空：
```
let possiblePlanet = Planet(rawValue: 7)
possiblePlanet?.rawValue
```

### 递归枚举
枚举成员关联值的类型也是当前枚举的话，则为递归枚举，此时必须使用`indirect`关键字：
```
enum ArithmeticExpression {
    case number(Int)
    indirect case addition(ArithmeticExpression, ArithmeticExpression)
    indirect case multiplication(ArithmeticExpression, ArithmeticExpression)
}

// 也可以在枚举之前写 indirect 来让整个枚举成员在需要时可以递归：
indirect enum ArithmeticExpression {
    case number(Int)
    case addition(ArithmeticExpression, ArithmeticExpression)
    case multiplication(ArithmeticExpression, ArithmeticExpression)
}
```
**但递归枚举中必然需要存在一个非递归的成员，因为死循环递归的成员是不可能创建出来的**，上面例子的`case number(Int)`就是这个非递归成员。

## 结构体和类
* 结构体和枚举都是值类型，类是引用类型
* Swift 中所有的基本类型——整数、浮点数、布尔量，以及字符串、数组和字典——都是值类型，并且都以结构体的形式在后台实现
* 引用同一个对象，使用===和!==判断

## 属性
* Swift的属性，区分了存储属性和计算属性，存储属性就是普通的实例属性，而计算属性就是setter和getter功能的属性。对比Kotlin，Swift的计算属性必须有一个直接或间接对应的存储属性，而Kotlin的属性有幕后字段，所以Kotlin的属性既可以是实例属性也可以是计算属性。
* willSet 和 didSet 可以作为属性观察者
* 属性包装

> 结构体、类、枚举都可以定义属性。但枚举只能使用计算属性。

`lazy` 用于延迟存储属性：
```
class DataManager {
    // importer 在第一次使用时才会被初始化
    lazy var importer = DataImporter()
}
```

### 计算属性
```
struct Point {
    var x = 0.0, y = 0.0
}
struct Size {
    var width = 0.0, height = 0.0
}
struct Rect {
    var origin = Point()
    var size = Size()
    var center: Point {
        get {
            let centerX = origin.x + (size.width / 2)
            let centerY = origin.y + (size.height / 2)
            return Point(x: centerX, y: centerY)
        }
        set(newCenter) {
            origin.x = newCenter.x - (size.width / 2)
            origin.y = newCenter.y - (size.height / 2)
        }
    }
}

// getter 可以省略 return，setter 可以省略参数，默认为 newValue
struct AlternativeRect {
    var origin = Point()
    var size = Size()
    var center: Point {
        get {
            Point(x: origin.x + (size.width / 2),
                  y: origin.y + (size.height / 2))
        }
        set {
            origin.x = newValue.x - (size.width / 2)
            origin.y = newValue.y - (size.height / 2)
        }
    }
}
```

只有 getter 则为只读计算属性，并且可以省略 get 关键字：
```
struct Cuboid {
    var width = 0.0, height = 0.0, depth = 0.0
    var volume: Double {
        return width * height * depth
    }
}
```

### 属性观察者
`willSet` 会在该值被存储之前被调用，`didSet` 会在一个新值被存储后被调用。
```
class StepCounter {
    var totalSteps: Int = 0 {
        willSet(newTotalSteps) {
            print("About to set totalSteps to \(newTotalSteps)")
        }
        didSet {
            if totalSteps > oldValue  {
                print("Added \(totalSteps - oldValue) steps")
            }
        }
    }
}
```
`willSet`如果没有写参数，默认会使用`newValue`作为参数名称；`didSet`如果没有写参数，默认会使用`oldValue`作为参数名称

### 属性包装
有一点类似Kotlin的属性委托，SwiftUI里面的`@State`等注解就利用了这个语法。属性包装中会使用`wrappedValue`这个关键字

```
// 限制属性的值范围不超过12
@propertyWrapper
struct TwelveOrLess {
    private var number = 0
    var wrappedValue: Int {
        get { return number }
        set { number = min(newValue, 12) }
    }
}

struct SmallRectangle {
    @TwelveOrLess var height: Int
    @TwelveOrLess var width: Int
}

rectangle.height = 24
print(rectangle.height)
// Prints "12"
```

#### 可以使用init初始化包装值
```
@propertyWrapper
struct SmallNumber {
    private var maximum: Int
    private var number: Int
 
    var wrappedValue: Int {
        get { return number }
        set { number = min(newValue, maximum) }
    }
 
    init() {
        maximum = 12
        number = 0
    }
    init(wrappedValue: Int) {
        maximum = 12
        number = min(wrappedValue, maximum)
    }
    init(wrappedValue: Int, maximum: Int) {
        self.maximum = maximum
        number = min(wrappedValue, maximum)
    }
}

// 给属性应用包装但并不指定初始值时，Swift 使用 init() 初始化器来设置包装：
struct ZeroRectangle {
    @SmallNumber var height: Int
    @SmallNumber var width: Int
}

// 为属性指定一个初始值时，Swift 使用 init(wrappedValue:) 初始化器来设置包装：
struct UnitRectangle {
    @SmallNumber var height: Int = 1
    @SmallNumber var width: Int = 1
}

// 在自定义特性后的括号中写实际参数时，Swift 使用接受那些实际参数的初始化器来设置包装：
struct NarrowRectangle {
    @SmallNumber(wrappedValue: 2, maximum: 5) var height: Int
    @SmallNumber(wrappedValue: 3, maximum: 4) var width: Int
}
```

#### 属性包装的映射值
属性包装器(Property Wrapper)的**映射值(projected value)**是一个额外的值，可以通过$前缀访问，它允许属性包装器提供除主值(wrapped value)之外的附加功能或信息。这是 SwiftUI 中广泛使用的模式（如 @State, @Binding 等）。

映射值的名称必须使用`projectedValue`
```
@propertyWrapper
struct SmallNumber {
    private var number = 0
    var projectedValue = false
    var wrappedValue: Int {
        get { return number }
        set {
            if newValue > 12 {
                number = 12
                projectedValue = true
            } else {
                number = newValue
                projectedValue = false
            }
        }
    }
}
struct SomeStructure {
    @SmallNumber var someNumber: Int
}
var someStructure = SomeStructure()
 
someStructure.someNumber = 4
print(someStructure.$someNumber)
```

### 类型属性
类似于Java的静态变量：
```
struct SomeStructure {
    static var storedTypeProperty = "Some value."
    static var computedTypeProperty: Int {
        return 1
    }
}
enum SomeEnumeration {
    static var storedTypeProperty = "Some value."
    static var computedTypeProperty: Int {
        return 6
    }
}
class SomeClass {
    static var storedTypeProperty = "Some value."
    static var computedTypeProperty: Int {
        return 27
    }
    class var overrideableComputedTypeProperty: Int {
        return 107
    }
}
```

## 方法
类，结构体以及枚举都能定义实例方法。每一个类的实例都隐含一个叫做 self的属性，它完完全全与实例本身相等

结构体和枚举是值类型，但可以使用`mutating`修饰的方法来修改自身的值（实际就是返回一个新的值，只是顺便帮我们赋值到原本的变量）。
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

### 类型方法
类似Java的静态方法：
```
class MyClass {
    // 使用 static 定义 - 不可被子类重写（静态派发）
    static func staticMethod() {
        print("Static method")
    }
    
    // 使用 class 定义 - 可以被子类重写（动态派发）
    class func classMethod() {
        print("Class method")
    }
}
```
> 结构体中，只能使用`static`关键字定义静态方法。


## 下标
下标语法，可以自定义`subscript`，类似计算属性拥有setter和getter，并且可以接收参数；使用时通过`[]`来获取返回值。
```
// 可读可写：
subscript(index: Int) -> Int {
    get {
        // Return an appropriate subscript value here.
    }
    set(newValue) {
        // Perform a suitable setting action here.
    }
}

// 只读：
struct TimesTable {
    let multiplier: Int
    subscript(index: Int) -> Int {
        return multiplier * index
    }
}
let threeTimesTable = TimesTable(multiplier: 3)
print("six times three is \(threeTimesTable[6])")
```

下标支持多维：
```
struct Matrix {
    let rows: Int, columns: Int
    var grid: [Double]
    init(rows: Int, columns: Int) {
        self.rows = rows
        self.columns = columns
        grid = Array(repeating: 0.0, count: rows * columns)
    }
    func indexIsValid(row: Int, column: Int) -> Bool {
        return row >= 0 && row < rows && column >= 0 && column < columns
    }
    subscript(row: Int, column: Int) -> Double {
        get {
            assert(indexIsValid(row: row, column: column), "Index out of range")
            return grid[(row * columns) + column]
        }
        set {
            assert(indexIsValid(row: row, column: column), "Index out of range")
            grid[(row * columns) + column] = newValue
        }
    }
}

var matrix = Matrix(rows: 2, columns: 2)
matrix[0, 1] = 1.5
matrix[1, 0] = 3.2
```

### 类型下标
```
enum Planet: Int {
    case mercury = 1, venus, earth, mars, jupiter, saturn, uranus, neptune
    static subscript(n: Int) -> Planet {
        return Planet(rawValue: n)!
    }
}
let mars = Planet[4]
print(mars)
```


## 继承
Swift的继承，可以使用`override`关键字重写方法、属性、属性观察器，并可以用`final`阻止重写

## 初始化
Celsius 结构体实现了两个自定义的初始化器 init(fromFahrenheit:) 和 init(fromKelvin:) ， 它们用不同的温度单位初始化新的结构体实例：
```
struct Celsius {
    var temperatureInCelsius: Double
    init(fromFahrenheit fahrenheit: Double) {
        temperatureInCelsius = (fahrenheit - 32.0) / 1.8
    }
    init(fromKelvin kelvin: Double) {
        temperatureInCelsius = kelvin - 273.15
    }
}
let boilingPointOfWater = Celsius(fromFahrenheit: 212.0)
// boilingPointOfWater.temperatureInCelsius is 100.0
let freezingPointOfWater = Celsius(fromKelvin: 273.15)
// freezingPointOfWater.temperatureInCelsius is 0.0
```

### 参数标签
初始化函数和普通函数一样，都可以使用参数标签，或者使用下划线( _ )定义函数实现调用时声明参数标签
```
struct Celsius {
    var temperatureInCelsius: Double
    init(fromFahrenheit fahrenheit: Double) {
        temperatureInCelsius = (fahrenheit - 32.0) / 1.8
    }
    init(fromKelvin kelvin: Double) {
        temperatureInCelsius = kelvin - 273.15
    }
    init(_ celsius: Double) {
        temperatureInCelsius = celsius
    }
}
let bodyTemperature = Celsius(37.0)
// bodyTemperature.temperatureInCelsius is 37.0
```

* 结构体有存储属性，即使没有默认值，也可以不写`init`函数，因为会自动生成，在构造时就必须传入参数。而类的属性只要没有初始值（可选类型自动初始化为`nil`），就必须明确声明`init`函数。

### 便捷初始化器
Swift中的初始化器分为两种：
* 指定初始化器(Designated)：负责完全初始化所有属性并调用父类初始化器
* 便捷初始化器(Convenience)：只作为辅助初始化途径，必须最终委托给指定初始化器

便捷初始化器，使用`convenience`修饰init函数，强制要求当前init函数调用当前类（**类的规则，结构体不需要添加convenience**）中的其他init函数，也就是委托给其他init函数（不能委托给父类的init函数）。（在Java/Kotlin的语法中，这种情况是不需要额外关键字的）
```
class Food {
    var name: String
    
    // 指定初始化器
    init(name: String) {
        self.name = name
    }
    
    // 便捷初始化器
    convenience init() {
        self.init(name: "[Unnamed]")  // 必须委托给指定初始化器
    }
}

class RecipeIngredient: Food {
    var quantity: Int

    // 指定初始化器
    init(name: String, quantity: Int) {
        self.quantity = quantity
        super.init(name: name)
    }

    // 便捷初始化器
    override convenience init(name: String) {
        self.init(name: name, quantity: 1)
    }
}
```
> `RecipeIngredient`会从`Food`继承无参的`init()`函数，因为`Food`的`init()`委托调用了`init(name: String)`，刚好`RecipeIngredient`重写了`init(name: String)`，所以就可以自动继承`init()`。

### 可以不调用 super.init 的情况
* 当父类有无参数的默认初始化器（且子类没有新增存储属性）时
* 当父类是 NSObject 等有默认无参初始化器的类时
```
class Parent {
    // 自动获得默认 init() {}
}

class Child: Parent {
    // 可以不写 init，或直接继承父类无参 init
    init() {
        // 可以不调用 super.init()，编译器会自动添加
    }
}
```

### 继承的初始化器
* 子类init和父类init函数参数相同时，需要使用override表示重写。
* Swift的init函数中，初始化当前类的属性，必须在调用super.init之前，这和Java等语言不同。这是为了避免问题：子类重写父类的方法，在重写方法中访问自身的属性，而此方法在父类中会在init中调用，导致子类还没有初始化就访问了该属性。
* Java/Kotlin的类，只要父类没有`无参构造函数`，子类都必须声明自己的构造函数，并调用父类的`有参构造函数`，即使只是简单的包装调用。而Swift在子类没有定义任何init函数时，是可以自动继承父类init函数，而不用强制显式写init函数。

init函数继承规则：
1. 如果子类没有定义init函数，则自动继承父类的全部init函数，包括便捷初始化函数。
2. 如果子类实现了父类的全部指定初始化函数，则自动继承父类的全部便捷初始化函数。
3. 如果子类没有实现父类的任何指定初始化函数，或者只实现了部分指定初始化函数，则不会继承父类的其他任何初始化函数。
4. 不能像重写指定初始化器那样使用 `override` 关键字来重写便捷初始化器，因为便捷初始化器本质上只是辅助初始化途径，必须委托给当前类的指定初始化器。要达到类似重写的目的，子类只能定义新的便捷初始化函数，或者定义一个函数参数列表一样的指定初始化器。


* init?表示初始化可以失败，返回nil。
```
struct Animal {
    let species: String
    init?(species: String) {
        if species.isEmpty { return nil }
        self.species = species
    }
}
```
可失败初始化器也可以被重写：
```
class Document {
    var name: String?
    // this initializer creates a document with a nil name value
    init() {}
    // this initializer creates a document with a non-empty name value
    init?(name: String) {
        self.name = name
        if name.isEmpty { return nil }
    }
}

class AutomaticallyNamedDocument: Document {
    override init() {
        super.init()
        self.name = "[Untitled]"
    }
    override init(name: String) {
        super.init()
        if name.isEmpty {
            self.name = "[Untitled]"
        } else {
            self.name = name
        }
    }
}
```

### 必要初始化器
在init前添加`required`修饰符来表明所有该类的子类都必须实现该初始化器。

### 可以通过闭包（函数）来初始化属性。
```
class SomeClass {
    let someProperty: SomeType = {
        // create a default value for someProperty inside this closure
        // someValue must be of the same type as SomeType
        return someValue
    }()
}
// 注意闭包花括号的结尾跟一个没有参数的圆括号。这是告诉 Swift 立即执行闭包。如果你忽略了这对圆括号，你就会把闭包作为值赋给了属性，并且不会返回闭包的值。
```

## 反初始化
Swift采用引用计数法来释放内存，所以相对于Java系语言，反初始化的调用时机更可靠，也就更能实际使用。
```
class Player {
    var coinsInPurse: Int
    init(coins: Int) {
        coinsInPurse = Bank.distribute(coins: coins)
    }
    func win(coins: Int) {
        coinsInPurse += Bank.distribute(coins: coins)
    }
    deinit {
        Bank.receive(coins: coinsInPurse)
    }
}
```

## 可选链
Swift的可空操作，和Kotlin大同小异：
```
// Swift
let name = person?.name ?? ""
let age = person!.age

// Kotlin
val name = person?.name ?: ""
val age = person!!.age
```

有点特色的是Swift的`可选绑定（optional binding）`语法，以及可以对下标使用可空语法：
```
// Kotlin中需要使用 let、also 之类的作用域函数
if let roomCount = john.residence?.numberOfRooms {
    print("John's residence has \(roomCount) room(s).")
} else {
    print("Unable to retrieve the number of rooms.")
}

// Kotlin的可空操作只能调用函数/属性，Swift则可以调用下标访问
john.residence?[0] = Room(name: "Bathroom")
```

## 错误处理
Swift所有错误都是`Error`协议的子类型，并且要么处理要么抛出（和Java的受检异常类似）

```
func canThrowErrors() throws -> String

do {
    try expression
    statements
} catch pattern 1 {
    statements
} catch pattern 2 where condition {
    statements
} catch pattern 3, pattern 4 where condition {
    statements
} catch {
    statements
}
```

### 转换错误为可空类型
使用`try?`通过将错误转换为可选项来处理一个错误
```
func someThrowingFunction() throws -> Int {
    // ...
}
 
let x = try? someThrowingFunction()

// 等价于上面 x 变量的方式
let y: Int?
do {
    y = try someThrowingFunction()
} catch {
    y = nil
}
```

### 取消错误传递
Swift的错误也具有传递机制，即如果一个函数抛出错误，则调用者必须处理该错误。如果确定一个throw方法不会抛出错误，可以在表达式前写`try!`来取消错误传递并且把调用放进不会有错误抛出的运行时断言当中。如果错误真的抛出了，则得到一个运行时错误。
```
let photo = try! loadImage("./Resources/John Appleseed.jpg")
```

### 指定错误类型
```
func summarize(_ ratings: [Int]) throws(StatisticsError) {
    guard !ratings.isEmpty else { throw .noRatings }


    var counts = [1: 0, 2: 0, 3: 0]
    for rating in ratings {
        guard rating > 0 && rating <= 3 else { throw .invalidRating(rating) }
        counts[rating]! += 1
    }


    print("*", counts[1]!, "-- **", counts[2]!, "-- ***", counts[3]!)
}
```

### 清理操作
`defer`语句来在代码离开当前代码块前执行语句合集，也就是其他语言中的`finally`语句。
```
func processFile(filename: String) throws {
    if exists(filename) {
        let file = open(filename)
        defer {
            close(file)
        }
        while let line = try file.readline() {
            // Work with the file.
        }
        // close(file) is called here, at the end of the scope.
    }
}
```
`defer`可以写在方法内任何位置（只要在作用域结束前），但必须在 return/throw 之前，`defer`也可以在同一个方法里面写多次。

## 宏
宏会添加新的代码，但不会删除或修改现有的代码。Swift 有两种类型的宏：
* 独立宏（Freestanding Macros）：独立宏自己出现，没有附加到声明。
* 附加宏（Attached Macros）：附加宏修改它们所附加到的声明。

### 独立宏
要调用独立宏，你在宏名称前加上井号（#），并在名称后的括号中写入宏的任何参数：
```
func myFunction() {
    print("Currently running \(#function)")
    #warning("Something's wrong")
}
```
`#function`是Swift标准库中的宏，编译这段代码时，Swift 会调用该宏的实现，将`#function`替换为当前函数的名称。`#warning`调用了 Swift 标准库中的`warning(_:)` 宏，用于生成自定义的编译时警告。

### 附加宏
调用附加宏，你在宏名称前加上 at 符号（@），并在名称后的括号中写入宏的任何参数，附加宏会修改它们所附加到的声明

```
// 不使用宏的代码，需要重复且手动编写
struct SundaeToppings: OptionSet {
    let rawValue: Int
    static let nuts = SundaeToppings(rawValue: 1 << 0)
    static let cherry = SundaeToppings(rawValue: 1 << 1)
    static let fudge = SundaeToppings(rawValue: 1 << 2)
}

// 使用 Swift 标准库中的 @OptionSet 宏
@OptionSet<Int>
struct SundaeToppings {
    private enum Options: Int {
        case nuts
        case cherry
        case fudge
    }
}
```
> 附加宏有点类似Java中注解的部分作用

### 自定义宏
之后分析

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

```
// 协议中的属性只能使用 var，不能使用let，并且需要声明get、set表示可读、可写
protocol SomeProtocol {
    var mustBeSettable: Int { get set }
    var doesNotNeedToBeSettable: Int { get }
}


// 子类必须实现父类的所有初始化器，并且必须使用required修饰符
protocol SomeProtocol {
    init(someParameter: Int)
}

class SomeClass: SomeProtocol {
    required init(someParameter: Int) {
        // initializer implementation goes here
    }
}

// 定义 mutating 方法，对于结构体就会要求必须是mutating，对于类没有影响
protocol Togglable {
    mutating func toggle()
}
```

Swift中的协议，可以使用`Self`，来达到实现协议时，确定`Self`类型为当前类型，也算是泛型语法的一种。对比Kotlin和Swift：
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

### 协议组合
要求一个类型一次遵循多个协议是很有用的：
```
protocol Named {
    var name: String { get }
}
protocol Aged {
    var age: Int { get }
}
struct Person: Named, Aged {
    var name: String
    var age: Int
}
func wishHappyBirthday(to celebrator: Named & Aged) {
    print("Happy birthday, \(celebrator.name), you're \(celebrator.age)!")
}
let birthdayPerson = Person(name: "Malcolm", age: 21)
wishHappyBirthday(to: birthdayPerson)
// Prints "Happy birthday, Malcolm, you're 21!"
```

### 给协议扩展添加限制
可以给 Collection 定义一个扩展来应用于任意元素遵循 TextRepresentable 协议的集合
```
extension Collection where Iterator.Element: TextRepresentable {
    var textualDescription: String {
        let itemsAsText = self.map { $0.textualDescription }
        return "[" + itemsAsText.joined(separator: ", ") + "]"
    }
}
```


## 泛型
* 泛型可以使用`where`提供更多约束
* 协议中使用泛型，是使用`associatedtype`语法
* 泛型协议（使用关联类型，或者`Self`）不能作为类型来修饰变量、返回值等。因为关联类型不确定，只有被子类实现后才是确定的。

泛型的类型参数由调用者确定，`associatedtype`关联类型由实现者决定。Swift在协议中不使用尖括号语法而是关联类型来表达泛型的具体原因待探究。

### 泛型函数
```
func swapTwoValues<T>(_ a: inout T, _ b: inout T) {
    let temporaryA = a
    a = b
    b = temporaryA
}
```

### 泛型类型
泛型类型可以用于任意类型的自定义类、结构体、枚举：
```
// 泛型结构体
struct Stack<Element> {
    var items = [Element]()
    mutating func push(_ item: Element) {
        items.append(item)
    }
    mutating func pop() -> Element {
        return items.removeLast()
    }
}
```

### 类型约束
```
func someFunction<T: SomeClass, U: SomeProtocol>(someT: T, someU: U) {
    // function body goes here
}
```

### 关联类型
Swift的协议的泛型不能像类一样使用泛型参数，而是要使用 `associatedtype`：
```
protocol Container {
    associatedtype ItemType
    mutating func append(_ item: ItemType)
    var count: Int { get }
    subscript(i: Int) -> ItemType { get }
}
```

### 泛型下标
```
extension Container {
    subscript<Indices: Sequence>(indices: Indices) -> [Item]
        where Indices.Iterator.Element == Int {
            var result = [Item]()
            for index in indices {
                result.append(self[index])
            }
            return result
    }
}
```

## 不透明类型
`some`关键字，限制函数返回泛型协议的某一种确定的实现类型，编译器知道类型是确定的，所以可以通过编译。并且使用`some`后，可知调用该函数返回的类型是确定的一种，也就能确定多次调用该函数返回的是同一类型，可安全使用相互赋值等操作。

```
protocol Shape {
    func draw() -> String
}

struct Circle: Shape {
    func draw() -> String { "Circle" }
}

struct Square: Shape {
    func draw() -> String { "Square" }
}

// 如果确定只会固定返回一种子类型，可以使用 some 关键字
func makeShape1() -> some Shape {
    return Circle() // 返回的具体类型是 Circle，但对外只暴露 Shape
}

// 如果返回多种类型，就不能使用 some 关键字
func makeShape2() ->  Shape {
    if Bool.random() {
            return Circle() // OK
        } else {
            return Square() // 错误：类型不一致
        }
}
```
返回`Shape`等价于`any Shape`。

使用`some`，编译器在编译时知道具体返回类型，可以进行内联和优化，避免运行时动态分发的开销。相比 any（存在类型）的动态分发，some 的性能更高。

## 自动引用计数
```
var reference1: Person?
var reference2: Person?

reference1 = Person(name: "John Appleseed")
reference2 = reference1 // 此时存在两个强引用



reference1 = nil // 留下一个强引用，Person 实例不会被销毁
reference2 = nil // 没有强引用， ARC 会销毁 Person 实例
```
引用计数仅适用于类的实例。结构和枚举是值类型，而不是引用类型，不按引用存储和传递。

> 引用计数变为0时，对象会立即被释放

> 在 Swift 中，一般不需要开发者主动使用autoreleasepool，但在与 Objective-C 代码交互或处理某些 Foundation 框架的 API，这些 API可能返回 autorelease 对象（例如 NSString、NSArray 等），则要考虑是否需要使用 autoreleasepool 来确保这些对象在正确的时间被释放。

### 循环强引用
Swift 提供了两种办法用来解决你在使用类的属性时所遇到的循环强引用问题：`弱引用（ weak reference ）`和`无主引用（ unowned reference）`。

* `weak`弱引用的类型必须是可空。建议在假定引用可能先于自身释放的情况下使用。
* `unowned`无主引用的类型不限制是否可空，更推荐使用非空。建议在可以保证引用比自身生命周期长的情况使用，也就是只要引用被释放，自身就不可能再被使用的情况，释放后再访问会导致崩溃。比如消费者类和银行卡类，消费者类的生命周期比银行卡类长，消费者类释放后不会再使用银行卡类对象，所以银行卡类持有消费者类对象时可以使用无主引用。

还可以使用`隐式展开`配合`unowned`无主引用：
```
class Country {
    let name: String
    var capitalCity: City! // 隐式展开，默认为nil，但访问时会直接获取展开的值
    init(name: String, capitalName: String) {
        self.name = name
        self.capitalCity = City(name: capitalName, country: self)
    }
}
 
class City {
    let name: String
    unowned let country: Country
    init(name: String, country: Country) {
        self.name = name
        self.country = country
    }
}

var country = Country(name: "Canada", capitalName: "Ottawa")
```


* 闭包如果捕获了当前类的属性（比如使用self），并且当前类的某个属性引用了此闭包，那么也会造成循环强引用。可以使用捕获列表来解决：
```
lazy var someClosure: (Int, String) -> String = {
    [unowned self] (index: Int, stringToProcess: String) -> String in
    // closure body goes here
}
```
> 捕获列表可以有多个item，并且可以使用 unowned，也可以使用 weak，根据需要选择适合的

### 闭包对引用计数的影响
```
class Task {
    var title = "Task 1"
    
    func start() {
        // 闭包捕获 self（强引用）
        let closure = {
            print("Running task: \(self.title)") // 增加 self 的引用计数
        }
        
        // 模拟异步执行（GCD）
        DispatchQueue.global().asyncAfter(deadline: .now() + 1) {
            closure()
        }
    }
    
    deinit {
        print("Task released")
    }
}

// 测试
var task: Task? = Task()
task?.start()
task = nil // 不会释放，因为闭包强引用
```
这个例子会等到1秒后执行完才会释放Task对象，因为闭包捕获了self，导致self的引用计数增加，异步任务执行完后释放闭包，从而才会释放Task对象。

如果闭包使用`weak`捕获列表，则不会增加self的引用计数：
```
class Task {
    var title = "Task 1"
    
    func start() {
        // 闭包捕获 self（强引用）
        let closure = { [weak self] in
            print("xxx Running task: \(self?.title ?? "Task released")") // 增加 self 的引用计数
        }
        
        // 模拟异步执行（GCD）
        DispatchQueue.global().asyncAfter(deadline: .now() + 2) {
            // 虽然这也是个闭包，但没有捕获 self，所以不会增加引用计数
            closure()
        }
    }
    
    deinit {
        print("xxx Task released")
    }
}

var task: Task? = Task()
task?.start()
task = nil // 可以立即释放，因为闭包弱引用
```

#### 闭包捕获Optional的场景
```
class ViewController: ViewController {
    override func viewDidLoad() {
        super.viewDidLoad()

        var person: Person? = Person(name: "张三")
        let closure = {
            print("捕获的人名: \(person?.name)") // 如果使用 person!.name 会崩溃
        }
        
        print("person 设为 nil")
        person = nil

        print("调用闭包")
        closure()
    }
}

class Person {
    var name: String
    init(name: String) { self.name = name }
    deinit { print("\(name) 被释放") }
}

打印结果为：
person 设为 nil
张三 被释放
调用闭包
捕获的人名: nil
```
这个场景，看起来闭包捕获了person，person 设为 nil后，应该不会释放person，但实际上 person 被释放了。**因为闭包捕获的是 person 变量（一个 Optional<Person>），而不是 Person 实例本身，闭包并没有直接持有 Person 实例的强引用，因此 Person 实例的引用计数不会因为闭包而增加**

引用计数变化过程：
1. 创建person和闭包后，闭包强引用`Optional<Person>`实例（也就是person变量），person变量强引用Person实例，Person实例的引用计数为1
2. 设置person为nil，导致Person实例的引用计数为0，闭包对`Optional<Person>`实例的引用计数仍然为1，`Optional<Person>`实例还在，而Person实例会被释放。

处理方式：

1. 解包，引用Person实例
```
var person: Person? = Person(name: "张三")
guard let strongPerson = person else { return } // 解包
// 或者 let strongPerson = person!

let closure = {
    print("捕获的人名: \(strongPerson.name)") // 捕获 strongPerson
}
person = nil
closure()
```
这样就可以在闭包执行后，闭包被释放，才会释放Person实例

2. 捕获列表显式强引用
```
var person: Person? = Person(name: "张三")

let closure = { [person] in
    print("捕获的人名: \(person?.name)")
}
person = nil
closure()
```
明确捕获person，person是`Optional<Person>`类型，而`Optional`是枚举也就是值类型，在使用了捕获列表的情况下，值类型被捕获的是拷贝，所以即使person设为nil，闭包引用的`Optional`实例拷贝还拥有一个对Person的强引用，从而不会导致Person实例被释放，等到闭包被释放后才会释放Person实例。

## 访问控制
Swift 为代码的实体提供个五个不同的访问级别。这些访问级别和定义实体的源文件相关，并且也和源文件所属的模块相关。
* Open 和 Public：可以被任意源文件访问。
* Internal：可以被同一个模块的源文件访问。（默认）
* File-private：可以被当前源文件内访问。
* Private：将实体的使用限制于封闭声明。


getter 和 setter 可以分别设置访问级别：
```

struct TrackedString {
        // 外部只能读取，内部可写可读 
         private(set) var numberOfEdits = 0
         var value: String = "" {
         didSet {
               numberOfEdits += 1
         }
         }
}
```

## 支持运算符重载
为`Vector2D`类型重载加法运算：
```
struct Vector2D {
    var x = 0.0, y = 0.0
}
 
extension Vector2D {
    static func + (left: Vector2D, right: Vector2D) -> Vector2D {
        return Vector2D(x: left.x + right.x, y: left.y + right.y)
    }
}

let vector = Vector2D(x: 3.0, y: 1.0)
let anotherVector = Vector2D(x: 2.0, y: 4.0)
let combinedVector = vector + anotherVector
```

Swift的运算符重载支持前缀、中缀、后缀

## 语法
介绍一些场景的语法特性，可以在官方文档的语言引用部分查看：https://docs.swift.org/swift-book/documentation/the-swift-programming-language/aboutthelanguagereference

### Key-Path表达式
和反射有点类似（标准库有`Mirror`用于发射）：
```
struct SomeStructure {
    var someValue: Int
}

let s = SomeStructure(someValue: 12)
let pathToProperty = \SomeStructure.someValue

let value = s[keyPath: pathToProperty]
// value is 12
```

Key-Path 可以与泛型结合，用于抽象属性访问逻辑：
```
func getValue<T, V>(_ object: T, keyPath: KeyPath<T, V>) -> V {
    return object[keyPath: keyPath]
}
```

### selector表达式
Selector 表达式使用 #selector 语法，引用一个方法（Objective-C 方法或标记为 @objc 的 Swift 方法）的名称，生成一个 Selector 类型的值。Selector 本质上是 Objective-C 运行时中的方法标识符（SEL 类型），Swift 的 Selector 结构体是对其的封装。
```
import UIKit

class ViewController: UIViewController {
    @objc func buttonTapped(_ sender: UIButton) {
        print("Button was tapped!")
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let button = UIButton(type: .system)
        button.setTitle("Tap Me", for: .normal)
        // 使用 #selector 注册事件
        button.addTarget(self, action: #selector(buttonTapped(_:)), for: .touchUpInside)
        view.addSubview(button)
    }
}
```

### Key-Path字符串表达式
Key-Path 字符串表达式使用 #keyPath 语法，引用一个属性的路径（Objective-C 属性或标记为 @objc 的 Swift 属性），生成一个表示该路径的字符串。用于 KVC/KVO 等 Objective-C 运行时 API。
```
#keyPath(Person.name)          // 生成: "name"
#keyPath(Person.address.city)  // 生成: "address.city"
```

### 条件编译块
```
#if <#compilation condition#>
    <#statements#>
#endif
```

### precedencegroup
定义运算符的优先级和结合性

### 可用性检查
```
if #available(<#platform name#> <#version#>, <#...#>, *) {
    <#statements to execute if the APIs are available#>
} else {
    <#fallback statements to execute if the APIs are unavailable#>
}

if #unavailable(<#platform name#> <#version#>, <#...#>) {
    <#fallback statements to execute if the APIs are unavailable#>
} else {
    <#statements to execute if the APIs are available#>
}
```

### Attributes
属性语法，常见的比如`@main`、`@available`、`@objc`、`@dynamicMemberLookup`
```
@<#attribute name#>
@<#attribute name#>(<#attribute arguments#>)
```
> 附加的宏和属性包装器也使用特性语法
