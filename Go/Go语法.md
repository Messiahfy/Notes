参考资料：
* https://golang-china.github.io/gopl-zh/
* https://tour.go-zh.org/
* https://go.dev/doc/effective_go
* https://go.dev/ref/spec
* 如何编写 go 代码：https://go.dev/doc/code

# 基本
## 变量
Go语言主要有四种类型的声明语句：`var`、`const`、`type`和`func`，分别对应变量、常量、类型和函数实体对象的声明

完整的变量声明语法： var 变量名字 类型 = 表达式，类型和赋值都可以省略其中一个
```go
var i int = 1

var i, j, k int // 多个 int 变量声明
var b, f, s = true, 2.3, "four" // bool, float64, string

// 简短变量声明
a := 1
相当于
var a int = 1
```

### 指针
```go
func main() {
	i, j := 42, 2701

	p := &i         // 指向 i
	fmt.Println(*p) // 通过指针读取 i 的值
	*p = 21         // 通过指针设置 i 的值
	fmt.Println(i)  // 查看 i 的值

	p = &j         // 指向 j
	*p = *p / 37   // 通过指针对 j 进行除法运算
	fmt.Println(j) // 查看 j 的值
}
```

在Go语言中，返回函数中局部变量的地址也是安全的。（每次调用f函数都将返回不同的结果，因为是不同地址的变量）
```go
var p = f()

func f() *int {
    v := 1
    return &v
}
```

### new函数
另一个创建变量的方法是调用内建的`new`函数。表达式`new(T)`将创建一个T类型的匿名变量，初始化为T类型的零值，然后返回变量地址，返回的指针类型为`*T`。

```go
p := new(int)   // p, *int 类型, 指向匿名的 int 变量
fmt.Println(*p) // "0"
*p = 2          // 设置 int 匿名变量的值为 2
fmt.Println(*p) // "2"
```
用`new`创建变量和普通变量声明语句方式创建变量实际没有什么区别，换言之，new函数类似是一种语法糖，而不是一个新的基础概念。
```go
// 这两个newInt函数有着相同的行为

func newInt() *int {
    return new(int)
}

func newInt() *int {
    var dummy int
    return &dummy
}
```
`new`函数使用通常相对比较少，因为对于结构体来说，直接用字面量语法创建新变量的方法会更灵活

### 变量的生命周期
对于在包一级声明的变量来说，它们的生命周期和整个程序的运行周期是一致的。局部变量的生命周期则是动态的：每次从创建一个新变量的声明语句开始，直到该变量不再被引用为止，然后变量的存储空间可能被回收。函数的参数变量和返回值变量都是局部变量。

**编译器会自动选择在栈上还是在堆上分配局部变量的存储空间，但可能令人惊讶的是，这个选择并不是由用var还是new声明变量的方式决定的。**
```go
var global *int

func f() {
    var x int
    x = 1
    global = &x
}

func g() {
    y := new(int)
    *y = 1
}
```
f函数里的x变量必须在堆上分配，因为它在函数退出后依然可以通过包一级的global变量找到，即使它是在函数内部定义的。相反，当g函数返回时，变量*y将是不可达的，也就是说可以马上被回收的。因此，*y并没有从函数g中逃逸，编译器可以选择在栈上分配*y的存储空间（译注：也可以选择在堆上分配，然后由Go语言的GC回收这个变量的内存空间），虽然这里用的是`new`方式。其实在任何时候，你并不需为了编写正确的代码而要考虑变量的逃逸行为，要记住的是，逃逸的变量需要额外分配内存，同时对性能的优化可能会产生细微的影响。

Go语言的自动垃圾收集器也是可达性算法

> Go语言的内存管理与C++等语言不同，没有显式的“栈分配”或“堆分配”关键字，程序员不能直接强制指定栈或堆分配，而是由编译器自动优化。（可以通过代码结构间接影响）

## 类型
Go 使用`type`关键字定义类型，比如后面介绍的定义结构体、接口等类型都会使用`type`，类型别名也使用这个方式。 
```
type 类型名字 底层类型
```

## 包和文件
Go语言中的包和其他语言的库或模块的概念类似，目的都是为了支持模块化、封装、单独编译和代码重用。


包还可以让我们通过控制哪些名字是外部可见的来隐藏内部实现信息。在Go语言中，一个简单的规则是：如果一个名字是大写字母开头的，那么该名字是导出的

导入语句中类似"gopl.io/ch2/tempconv"的字符串对应包的导入路径。Go语言的规范并没有定义这些字符串的具体含义或包来自哪里，它们是由构建工具来解释的。当使用Go语言自带的go工具箱时（第十章），一个导入路径代表一个目录中的一个或多个Go源文件。


包的初始化，会根据包级变量的声明出现顺序和依赖顺序，依次初始化
```go
var a = b + c // a 第三个初始化, 为 3
var b = f()   // b 第二个初始化, 为 2, 通过调用 f (依赖c)
var c = 1     // c 第一个初始化, 为 1

func f() int { return c + 1 }
```

每个源文件都可以通过定义自己的无参数`init`函数来设置一些必要的状态。 （其实每个文件都可以拥有多个 init 函数。）
```go
func init() {
	if user == "" {
		log.Fatal("$USER not set")
	}
	if home == "" {
		home = "/home/" + user
	}
	if gopath == "" {
		gopath = home + "/go"
	}
	// gopath 可通过命令行中的 --gopath 标记覆盖掉。
	flag.StringVar(&gopath, "gopath", gopath, "override default GOPATH")
}
```

包的初始化顺序会根据包的依赖关系，main包最后被初始化

## 字符串
字符串是一个不可改变的字节序列，赋值和追加字符串都会引用新的字符串值，而不会修改原本的值
```go
s := "hello, world"
t := s
s += ", right foot"
```

## 常量
常量表达式的值在编译期计算，而不是在运行期。
```go
const pi = 3.14159

// 也可以一次声明多个常量，这比较适合声明一组相关的常量
const (
    e  = 2.71828182845904523536028747135266249775724709369995957496696763
    pi = 3.14159265358979323846264338327950288419716939937510582097494459
)
```

### iota 常量生成器
常量声明可以使用iota常量生成器初始化，它用于生成一组以相似规则初始化的常量，但是不用每行都写一遍初始化表达式。在一个const声明语句中，在第一个声明的常量所在的行，iota将会被置为0，然后在每一个有常量声明的行加一。

比如用来实现枚举：
```go
type Weekday int

const (
    Sunday Weekday = iota
    Monday
    Tuesday
    Wednesday
    Thursday
    Friday
    Saturday
)
```

下面是一个更复杂的例子，每个常量都是1024的幂：
```go
const (
    _ = 1 << (10 * iota)
    KiB // 1024
    MiB // 1048576
    GiB // 1073741824
    TiB // 1099511627776             (exceeds 1 << 32)
    PiB // 1125899906842624
    EiB // 1152921504606846976
    ZiB // 1180591620717411303424    (exceeds 1 << 64)
    YiB // 1208925819614629174706176
)
```

# 复合数据类型
## 数组
因为数组的长度是固定的，因此在Go语言中很少直接使用数组。和数组对应的类型是`Slice`（切片），它是可以增长和收缩的动态序列，slice功能也更灵活，但是要理解slice工作原理的话需要先理解数组。
```go
var a [2]string
a[0] = "Hello"
a[1] = "World"

for i, v := range a {
    fmt.Printf("%d %d\n", i, v)
}

for _, v := range a {
    fmt.Printf("%d\n", v)
}
```
数组可以使用`range`遍历每次循环迭代，`range`产生一对值：索引以及在该索引处的元素值

默认情况下，数组的每个元素都被初始化为元素类型对应的零值，对于数字类型来说就是0。我们也可以使用数组字面值语法用一组值来初始化数组：
```go
var q [3]int = [3]int{1, 2, 3}
var r [3]int = [3]int{1, 2}
fmt.Println(r[2]) // "0"

// 如果在数组的长度位置出现的是“...”省略号，则表示数组的长度是根据初始化值的个数来计算
q := [...]int{1, 2, 3}
```

数组还可以指定一个索引和对应值列表的方式初始化，就像下面这样：
```go
type Currency int

const (
    USD Currency = iota // 美元
    EUR                 // 欧元
    GBP                 // 英镑
    RMB                 // 人民币
)

symbol := [...]string{USD: "$", EUR: "€", GBP: "￡", RMB: "￥"}
fmt.Println(RMB, symbol[RMB]) // "3 ￥"


// 定义了一个含有100个元素的数组r，最后一个元素被初始化为-1，其它元素都是用0初始化。
r := [...]int{99: -1}
```

如果数组的元素类型可比较，则可以直接通过`==`和`!=`比较运算符来比较两个数组，当两个数组的所有元素都是相等的时候数组才是相等的，不相等比较运算符遵循同样的规则。


* go 数组是值，将一个数组赋予另一个数组会复制其所有元素，所以一般使用指针（但更多会使用`slice`）
* 数组的大小是其类型的一部分。类型 [10]int 和 [20]int 是不同的。

## Slice
先了解一下`Slice`的核心概念，数组的大小都是固定的，而`slice`是可变的。`slice`只是数组的`View`，它的底层会引用一个数组对象。一个slice由三个部分构成：指针、长度和容量。
1. 指针指向第一个`slice`元素对应的底层数组元素的地址，要注意的是`slice`的第一个元素并不一定就是数组的第一个元素。
2. 长度对应`slice`中元素的数目
3. 长度不能超过容量，容量一般是从`slice`的开始位置到底层数据的结尾位置。
内置的`len`和`cap`函数分别返回`slice`的长度和容量。

类型`[]T`表示一个元素类型为`T`的 slice（和数组类型的表示相似，但数组需要指定长度，比如`[3]int`），slice 通过两个下标来界定，它会选出一个半闭半开区间，包括第一个元素，但排除最后一个元素。
```go
primes := [6]int{2, 3, 5, 7, 11, 13}
var s []int = primes[1:4] // s == [3, 5, 7]
```

slice 并不存储任何数据，它只是描述了底层数组中的一段，更改 slice 的元素会修改其底层数组中对应的元素。多个 slice 之间可以共享底层的数据，并且引用的数组部分区间可能重叠，那么和它共享底层数组的切片都会观测到这些修改。
```go
names := [4]string{
		"John",
		"Paul",
		"George",
		"Ringo",
	}

a := names[0:2]
b := names[1:3]

b[0] = "XXX" // 切片a也会受到影响
```

切片下界的默认值为 0，上界则是该切片的长度。
```go
var a [10]int
```
对于数组来说，以下切片表达式和它是等价的：
```go
a[0:10]
a[:10]
a[0:]
a[:]
```

### slice 字面量
切片字面量类似于没有长度的数组字面量。

这是一个数组字面量：
```go
[3]bool{true, true, false}
```

下面这样则会创建一个和上面相同的数组，然后再构建一个引用了它的切片：
```go
[]bool{true, true, false}
```

### 用 make 创建 slice
在底层，make创建了一个匿名的数组变量，然后返回一个slice
```go
a := make([]int, 5)  // len(a)=5  自动创建底层数组[5]int{}

b := make([]int, 0, 5) // len(b)=0, cap(b)=5

b = b[:cap(b)] // len(b)=5, cap(b)=5
b = b[1:]      // len(b)=4, cap(b)=4
```

### 切片增删
内置的`append`函数用于向slice追加元素：
```go
var s []int
printSlice(s)

// 可在空切片上追加
s = append(s, 0)

// 可以一次性添加多个元素
s = append(s, 2, 3, 4)
```
切片会关联一个数组，当Go切片超出容量(cap)时，会创建一个新的底层数组，此时切片与原数组不再有任何关联。（类似 Java 的 ArrayList 自动扩容逻辑）

Slice 没有内置`remove`函数，只能自行操作，比如：
```go
func remove(slice []int, i int) []int {
    copy(slice[i:], slice[i+1:])
    return slice[:len(slice)-1]
}
``` 

## Map
```go
var m map[string]Vertex
// 使用内置的 make 函数可以创建一个map
m = make(map[string]Vertex)
m["Bell Labs"] = Vertex{
	40.68433, -74.39967,
}

// 使用字面量直接创建
var m = map[string]Vertex{
	"Bell Labs": Vertex{
		40.68433, -74.39967,
	},
	"Google": Vertex{
		37.42202, -122.08408,
	},
}

// 可以省略类型
var m = map[string]Vertex{
	"Bell Labs": {40.68433, -74.39967},
	"Google":    {37.42202, -122.08408},
}
```

Map 的操作方式：
```go
m[key] = elem // 插入或修改

elem = m[key] // 读取（如果一个查找失败将返回value类型对应的零值，比如int的0，string的""）

delete(m, key) // 删除

elem, ok = m[key] // 检测key是否存在，若 key 在 m 中，ok 为 true ；否则，ok 为 false。（可以区分是不存在所以是零值，还是确实就是零值）

elem, ok := m[key] // 若 elem 或 ok 还未声明，你可以使用短变量声明

// += 和 ++ 等简短赋值语法也可以用在map上
ages["bob"] += 1
ages["bob"]++
```

遍历：
```go
for name, age := range ages {
    fmt.Printf("%s\t%d\n", name, age)
}
```

## 没有 Set
简单情况一般通过`map[T]struct{}`或者`map[T]bool`模拟`Set`：
```go
// 创建
set := make(map[string]struct{})

// 添加
set["apple"] = struct{}{}

// 判断存在
if _, ok := set["apple"]; ok {
    fmt.Println("存在")
}
```

或者使用开源库 https://github.com/deckarep/golang-set

## 结构体
```go
type Vertex struct {
	X int
	Y int
}

var v = Vertex{1, 2}
v.X = 4
```

**通过指针操作结构体，支持隐式解引用（非结构体类型不支持自动解引用）**
```go
v := Vertex{1, 2}
p := &v
// 两种语法都可以
p.X = 1e9
(*p).X = 123
```

结构体字面量：
```go
var (
	v1 := Vertex{1, 2}  // 创建一个 Vertex 类型的结构体
	v2 := Vertex{X: 1}  // Y:0 被隐式地赋予零值
	v3 := Vertex{}      // X:0 Y:0
	p  := &Vertex{1, 2} // 创建一个 *Vertex 类型的结构体（指针）
)

// 结构体成员定义的顺序为每个结构体成员指定一个字面值，需要匹配顺序，一般只在定义结构体的包内部使用，或者是在较小的结构体中使用
// 更常用的是第二种写法，以成员名字和相应的值来初始化，可以包含部分或全部的成员
v := Vertex{X: 1, Y: 2}
```

> 考虑到效率，较大的结构体通常会用指针的方式传入和返回。如果要在函数内部修改结构体成员的话，用指针传入是必须的。

### 结构体嵌入（类似继承）
Go 采用了不同的设计哲学，使用**组合（Composition）**而不是继承来实现代码复用和扩展。
```go
// 派生结构体（类似子类）
type Dog struct {
    Animal      // 嵌入 Animal 结构体
    Breed string // Dog 特有的字段
}
```

### 结构体比较
如果结构体的全部成员都是可以比较的，那么结构体也是可以比较的，那样的话两个结构体将可以使用==或!=运算符进行比较。相等比较运算符==将比较两个结构体的每个成员，因此下面两个比较的表达式是等价的：
```go
type Point struct{ X, Y int }

p := Point{1, 2}
q := Point{2, 1}
fmt.Println(p.X == q.X && p.Y == q.Y) // "false"
fmt.Println(p == q)                   // "false"
```

### 结构体嵌套的匿名成员
像如下代码，嵌套的结构体，访问成员稍微有点繁琐：
```go
type Point struct {
    X, Y int
}

type Circle struct {
    Center Point
    Radius int
}

type Wheel struct {
    Circle Circle
    Spokes int
}

// 需要多个层级调用访问
var w Wheel
w.Circle.Center.X = 8
w.Circle.Center.Y = 8
w.Circle.Radius = 5
w.Spokes = 20
```
Go语言有一个特性让我们只声明一个成员对应的数据类型而不指名成员的名字，即匿名成员
```go
type Circle struct {
    Point
    Radius int
}

type Wheel struct {
    Circle
    Spokes int
}

// 这样就可以直接访问嵌套的内部成员
var w Wheel
w.X = 8            // equivalent to w.Circle.Point.X = 8
w.Y = 8            // equivalent to w.Circle.Point.Y = 8
w.Radius = 5       // equivalent to w.Circle.Radius = 5
w.Spokes = 20
```
### 结构体标签
```go
type User struct {
    Name string `json:"name" db:"user_name"`
    Age  int    `json:"age"`
}
```
* json:"name" 表示在 JSON 编解码时，这个字段对应的 key 是 name。
* db:"user_name" 表示数据库操作时，这个字段对应的列名是 user_name。

可以通过反射获取标签：
```go
import "reflect"

func main() {
    t := reflect.TypeOf(User{})
    field, _ := t.FieldByName("Name")
    tag := field.Tag.Get("json")
    fmt.Println(tag) // 输出: name
}
```

# 流程控制
## 1. for

Go的 for 循环类似于C，但却不尽相同。它统一了 for 和 while，不再有 do-while 了。它有三种形式，但只有一种需要分号。
```go
// 没有while，只有for
for sum < 1000 {
		sum += sum
	}

// 无限循环
	for {
	}
```

## 2. if

if 语句可以在条件之前执行一个简单的语句
```go
func pow(x, n, lim float64) float64 {
	if v := math.Pow(x, n); v < lim {
		return v
	} else {
		fmt.Printf("%g >= %g\n", v, lim)
	}
	// can't use v here, though
	return lim
}
```

## 3. switch

switch 也可以执行语句
```go
	switch os := runtime.GOOS; os {
	case "darwin":
		fmt.Println("macOS.")
	case "linux":
		fmt.Println("Linux.")
	default:
		// freebsd, openbsd,
		// plan9, windows...
		fmt.Printf("%s.\n", os)
	}
```

无条件的 switch
```go
	t := time.Now()
	switch {
	case t.Hour() < 12:
		fmt.Println("早上好！")
	case t.Hour() < 17:
		fmt.Println("下午好！")
	default:
		fmt.Println("晚上好！")
	}
```

# 函数和方法
函数的声明语法（支持多返回值）：
```go
func name(parameter-list) (result-list) {
    body
}
```

函数也是值，它们可以像其他值一样传递。
```go
func compute(fn func(float64, float64) float64) float64 {
	return fn(3, 4)
}
```

匿名函数：
```go
strings.Map(func(r rune) rune { return r + 1 }, "HAL-9000")
```

可变参数：
```go
func sum(vals ...int) int {
    total := 0
    for _, val := range vals {
        total += val
    }
    return total
}

values := []int{1, 2, 3, 4}
fmt.Println(sum(values...))
```

## defer
在调用普通函数或方法前加上关键字`defer`，函数在最后才会执行`defer`后的函数调用，无论函数是正常结束还是由于panic导致的异常结束。defer语句经常被用于处理成对的操作，如打开、关闭、连接、断开连接、加锁、释放锁。
```go
func main() {
	defer fmt.Println("world")

	fmt.Println("hello")
}
```
> 和 swift 的一样。并且如果是多个defer，按照栈的先进后出的顺序执行

## panic 异常
Go的类型系统会在编译时捕获很多错误，但有些错误只能在运行时检查，如数组访问越界、空指针引用等。这些运行时错误会引起panic异常。

当panic异常发生时，程序会中断运行，并开始回溯Go程的栈，并立即执行`defer`函数，回溯到达Go协程栈的顶端，程序崩溃并输出日志信息。

## recover 捕获异常
一般不需要对`panic`异常做任何处理，但有些情况我们想从异常中恢复，可以在崩溃前做一些操作。比如当web服务器遇到不可预料的严重问题时，在崩溃前应该将所有的连接关闭；如果不做任何处理，会使得客户端一直处于等待状态。

调用`recover`将停止回溯过程，并返回传入`panic`异常信息。由于在回溯时只有被`defer`函数中的代码在运行，因此`recover`在`defer`的函数中才有效。
```go
func Parse(input string) (s *Syntax, err error) {
    defer func() {
        if p := recover(); p != nil {
            err = fmt.Errorf("internal error: %v", p)
        }
    }()
    // ...parser...
}
```
> 可以通过判断`recover()`的返回值确定panic异常类型，选择性地处理


## 方法
Go 没有类。不过你可以为类型定义方法。方法就是一类带特殊的 **接收者** 参数的函数。
```go
type Vertex struct {
	X, Y float64
}

func (v Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func main() {
	v := Vertex{3, 4}
	fmt.Println(v.Abs())
}
```

对应的相同功能的普通方法为：
```go
func Abs(v Vertex) float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func main() {
	v := Vertex{3, 4}
	fmt.Println(Abs(v))
}
```

### 指针类型的接收者
要修改参数的值，则需要使用指针。（结构体类型支持自动解引用，其他类型不支持）
```go
func (v *Vertex) Scale(f float64) {
	v.X = v.X * f
	v.Y = v.Y * f
}

v := Vertex{3, 4}
v.Scale(10)

// 对应的函数写法
func Scale(v *Vertex, f float64) {
	v.X = v.X * f
	v.Y = v.Y * f
}
Scale(&v, 10)
```
使用方法的写法，调用时更方便，因为 Go 会将语句 `v.Scale(5)` 解释为 `(&v).Scale(5)`，开发者既可以使用指针，也可以使用值。

使用指针接收者的原因有二：
1. 首先，方法能够修改其接收者指向的值。
2. 其次，这样可以避免在每次调用方法时复制该值。若值的类型为大型结构体时，这样会更加高效。

> reciever 可以为 nil，方法内部则需要自行判断

### 方法作为值
方法的调用也可以分为两步，比如`p.Distance`为一个绑定了接收者为`p`的方法值，之后使用不用再指定接收者。
```go
p := Point{1, 2}
q := Point{4, 6}

distanceFromP := p.Distance        // method value
fmt.Println(distanceFromP(q))      // "5"
var origin Point                   // {0, 0}
fmt.Println(distanceFromP(origin)) // "2.23606797749979", sqrt(5)
```

方法表达式可以把方法作为变量传递：
```go
type Point struct{ X, Y float64 }

func (p Point) Add(q Point) Point { return Point{p.X + q.X, p.Y + q.Y} }
func (p Point) Sub(q Point) Point { return Point{p.X - q.X, p.Y - q.Y} }

Point.Add // Add 方法
Point.Sub // Sub 方法
```

# 接口
接口类型的定义为一组方法签名，接口类型的变量可以持有任何实现了这些方法的值。

一个类型如果拥有一个接口需要的所有方法，那么这个类型就实现了这个接口。接口无需专门显式声明，也就没有`implements`关键字
```go
type Abser interface {
	Abs() float64
}

// *Vertex 实现了 Abser 接口
func (v *Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

type MyFloat float64

func (f MyFloat) Abs() float64 {
	if f < 0 {
		return float64(-f)
	}
	return float64(f)
}

func main() {
	var a Abser
	f := MyFloat(-math.Sqrt2)
	v := Vertex{3, 4}

    a = f  // a MyFloat 实现了 Abser
	a = &v // a *Vertex 实现了 Abser
}
```

##  接口组合（类似继承）
新的接口类型可以通过组合已有的接口来定义：
```go
package io
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Closer interface {
    Close() error
}

type ReadWriter interface {
    Reader
    Writer
}
type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

## 接口值
接口也是值。它们可以像其它值一样传递。接口值由两个部分组成，一个具体的类型和那个类型的值。
```go
type I interface {
	M()
}

type T struct {
	S string
}

func (t *T) M() {
	fmt.Println(t.S)
}

type F float64

func (f F) M() {
	fmt.Println(f)
}

func main() {
	var i I

	i = &T{"Hello"} // 实现 I 接口的是 *T 类型，所以这里使用指针
	describe(i)
	i.M()

	i = F(math.Pi)
	describe(i)
	i.M()
}

func describe(i I) {
	fmt.Printf("(%v, %T)\n", i, i)
}
```
使用接口类型声明的变量，可以引用任何符合此接口的值。

> %T 打印接口的具体类型、%v 打印接口的具体值

## nil
即便接口内的具体值为 nil，方法仍然会被 nil 接收者调用。在一些语言中，这会触发一个空指针异常，但在 Go 中通常会写一些方法来优雅地处理它（如本例中的 M 方法）。
```go
func (t *T) M() {
	if t == nil {
		fmt.Println("<nil>")
		return
	}
	fmt.Println(t.S)
}

func main() {
	var i I

	var t *T
	i = t
	i.M()
}
```

## 空接口
指定了零个方法的接口值被称为**空接口**，空接口可保存任何类型的值，相当于 any 类型。（因为每个类型都至少实现了零个方法）空接口被用来处理未知类型的值
```go
func main() {
	var i interface{}
	describe(i) // (<nil>, <nil>)

	i = 42
	describe(i) // (42, int)

	i = "hello"
	describe(i) // (hello, string)
}

func describe(i interface{}) {
	fmt.Printf("(%v, %T)\n", i, i)
}
```

## 类型断言
类型断言 提供了访问接口值底层具体值的方式。`t := i.(T)`，如果 i 确实是 T 类型，则会把 i 的值赋给 t，否则报错。如果使用 `t, ok := i.(T)`，则断言成功时，t 为赋值，ok 为 true，否则 t 为 T 类型的零值，ok 为 false，不会报错。
```go
func main() {
	var i interface{} = "hello"

	s := i.(string)
	fmt.Println(s) // hello

	s, ok := i.(string)
	fmt.Println(s, ok) // hello true

	f, ok := i.(float64)
	fmt.Println(f, ok) // 0 false

	f = i.(float64) // panic: interface conversion: interface {} is string, not float64
	fmt.Println(f)
}
```
可以看到这个写法，可以 map 的读取操作相同

## 类型选择
如果通过 if else 分支判断多种类型的情况，略微繁琐，可以使用type switch语法简化。类型选择中的声明与类型断言 i.(T) 的语法相同，只是具体类型 T 被替换成了关键字`type`。
```go
func do(i interface{}) {
	switch v := i.(type) {
	case int:
		fmt.Printf("二倍的 %v 是 %v\n", v, v*2)
	case string:
		fmt.Printf("%q 长度为 %v 字节\n", v, len(v))
	default:
		fmt.Printf("我不知道类型 %T!\n", v)
	}
}

func main() {
	do(21)
	do("hello")
	do(true)
}
```

## 例如：Stringer接口
fmt 包（还有很多包）都通过此接口来打印值。作用类似 Java 的 toString()方法
```go
type Stringer interface {
    String() string
}
```

例如：
```go
type Person struct {
	Name string
	Age  int
}

func (p Person) String() string {
	return fmt.Sprintf("%v (%v years)", p.Name, p.Age)
}

func main() {
	a := Person{"Arthur Dent", 42}
	z := Person{"Zaphod Beeblebrox", 9001}
	fmt.Println(a, z)
}
```

## 错误
Go 程序使用 error 值来表示错误状态。error 类型是一个内建接口：
```go
type error interface {
    Error() string
}
```

通常函数会返回一个 error 值，调用它的代码应当判断这个错误是否等于 nil 来进行错误处理。
```go
type DivError struct {
    A, B int
}

func (e *DivError) Error() string {
    return fmt.Sprintf("cannot divide %d by %d", e.A, e.B)
}

func Div(a, b int) (int, error) {
    if b == 0 {
        return 0, &DivError{a, b}
    }
    return a / b, nil
}

value, err := Div(4, 0)
if err != nil {
	// ...
}
```
当函数返回non-nil的error时，其他的返回值是未定义的（undefined），这些未定义的返回值应该被忽略。然而，有少部分函数在发生错误时，仍然会返回一些有用的返回值。比如，当读取文件发生错误时，Read函数会返回可以读取的字节数以及错误信息。对于这种情况，正确的处理方式应该是先处理这些不完整的数据，再处理错误。因此对函数的返回值要有清晰的说明，以便于其他人使用。

### 错误处理策略
```go
// 1. 对http.Get的调用失败，当前函数可以直接将这个HTTP错误返回给调用者：
resp, err := http.Get(url)
if err != nil{
    return nil, err
}

// 2. 或者增加一些信息：
doc, err := html.Parse(resp.Body)
resp.Body.Close()
if err != nil {
    return nil, fmt.Errorf("parsing %s as HTML: %v", url,err)
}

// 3. 结束程序
// 4. 或者并不返回错误，只是打印
// 5. 忽略错误
```

## IO
io 包指定了 io.Reader 接口，它表示数据流的读取端。

io.Reader 接口有一个 Read 方法：
```go
func (T) Read(b []byte) (n int, err error)
```

# 泛型
可以使用类型参数编写 Go 函数来处理多种类型。 函数的类型参数出现在函数参数之前的方括号之间。
```go
func Index[T comparable](s []T, x T) int
```
此声明意味着 s 是满足内置约束 comparable 的任何类型 T 的切片。 x 也是相同类型的值。

```go
// Index 返回 x 在 s 中的下标，未找到则返回 -1。
func Index[T comparable](s []T, x T) int {
	for i, v := range s {
		// v 和 x 的类型为 T，它拥有 comparable 可比较的约束，
		// 因此我们可以使用 ==。
		if v == x {
			return i
		}
	}
	return -1
}

func main() {
	// Index 可以在整数切片上使用
	si := []int{10, 20, 15, -10}
	fmt.Println(Index(si, 15))

	// Index 也可以在字符串切片上使用
	ss := []string{"foo", "bar", "baz"}
	fmt.Println(Index(ss, "hello"))
}
```

定义泛型类型
```go
// List 表示一个可以保存任何类型的值的单链表。
type List[T any] struct {
	next *List[T]
	val  T
}
```

# Go协程
https://www.youtube.com/watch?v=QDDwwePbDtw
https://vimeo.com/53221558


在Go语言中，每一个并发的执行单元叫作一个`goroutine`。当一个程序启动时，其主函数即在一个单独的goroutine中运行，我们叫它main goroutine。新的goroutine会用`go`语句来创建：
```go
go f(x, y, z) // f, x, y 和 z 的求值发生在当前的 Go 协程中，而 f 的执行发生在新的 Go 协程中。
```

简单例子：
```go
func say(s string) {
	for i := 0; i < 5; i++ {
		time.Sleep(100 * time.Millisecond)
		fmt.Println(s)
	}
}

func main() {
	go say("world")
	say("hello")
}
```

## Channel
Channel是带有类型的管道，你可以通过它用信道操作符 <- 来发送或者接收值，用于 goroutine 之间通信。

和 map、slice 一样，channel 在使用前必须创建，可以使用内置的 make 函数创建一个 channel：
```go
ch := make(chan int)
```
和map类似，channel也对应一个make创建的底层数据结构的引用。当我们复制一个channel或用于函数参数传递时，我们只是拷贝了一个channel引用，因此调用者和被调用者将引用同一个channel对象。

channel有发送和接受两个主要操作，都使用`<-`运算符，发送时`<-`放在channel和要发送的值之间，接收时`<-`放在channel之前。
```go
ch <- v    // 将 v 发送至信道 ch。
v := <-ch  // 从 ch 接收值并赋予 v。
<-ch // 仅接收数据，但不使用
```
“箭头”就是数据流的方向。

### 关闭channel
使用内置的`close`函数可以关闭 channel，关闭channel，随后对基于该channel的任何发送操作都将导致panic异常。对一个已经被close过的channel进行接收操作依然可以接受到之前已经成功发送的数据.
```go
close(ch)
```

没有办法直接测试一个channel是否被关闭，但是接收操作有一个变体形式：它多接收一个结果，多接收的第二个结果是一个布尔值ok，ture表示成功从channels接收到值，false表示channels已经被关闭并且里面没有值可接收。
```go
ch := make(chan int)

for {
        x, ok := <-ch
        if !ok {
            break // channel was closed and drained
        }
        fmt.Println(x)
    }
```
从 channel 接收数据，可以普通循环读取，也支持使用`range`遍历，当channel被关闭并且没有值可接收时跳出循环。
```go
ch := make(chan int)
for x := range ch {
    fmt.Println(x)
}
```

> 不管一个channel是否被关闭，当它没有被引用时将会被Go语言的垃圾自动回收器回收。

### channel缓存
以最简单方式调用make函数创建的是一个无缓存的channel，但是我们也可以指定第二个整型参数，对应channel的容量。如果channel的容量大于零，那么该channel就是带缓存的channel。
```go
ch = make(chan int)    // unbuffered channel
ch = make(chan int, 0) // unbuffered channel
ch = make(chan int, 3) // buffered channel with capacity 3
```

无缓存的channel发送操作会阻塞当前goroutine，直到另一个goroutine在相同的Channels上执行接收操作。默认情况下，发送和接收操作在另一端准备好之前都会阻塞。这使得 Go 程可以在没有显式的锁或竞态变量的情况下进行同步。

有缓存的channel的缓冲区填满后，向其发送数据时才会阻塞。当缓冲区为空时，接受方会阻塞

### 单方向的 channel
类型`chan<- int`表示一个只能向它发送int的channel，类型`<-chan int`表示一个只能从它接收int的channel。
```go
func testIn(in <-chan int) {
    for v := range in {
        fmt.Println(v)
    }
    close(out)
}

func testOut(out chan<- int) {
	out <- 123
}

func main() {
    inChannel := make(chan int)
    outChannel := make(chan int)
    go testIn(inChannel) // inChannel 的类型将隐式地从chan int转换成 <-chan int
    go testOut(outChannel) // outChannel 的类型将隐式地从chan int转换成 chan<- int
}
```

## select 多路复用
`select`语句使一个 Go 程可以等待多个channel操作。`select` 会阻塞到某个分支可以继续执行为止，这时就会执行该分支。当多个分支都准备好时会随机选择一个执行。

当 select 中的其它分支都没有准备好时，default 分支就会执行
```go
func main() {
	tick := time.Tick(100 * time.Millisecond)
	boom := time.After(500 * time.Millisecond)
	for {
		select {
		case <-tick:
			fmt.Println("tick.")
		case <-boom:
			fmt.Println("BOOM!")
			return
		default:
			fmt.Println("    .")
			time.Sleep(50 * time.Millisecond)
		}
	}
}
```

## goroutine退出
Go 语言没有提供类似 Kotlin 的 `Job.cancel()` 这种明确的方法来取消 goroutine，所以可以选择通过关闭 channel 的方式，关闭 Channel 后，所有从该 Channel 读取的 Goroutine 都会立即收到“零值”。
```go
var done = make(chan struct{})

// 检测到用户输入，就关闭 channel，其他监听 done 的 goroutine 就会收到零值
go func() {
    os.Stdin.Read(make([]byte, 1))  // 读取一个字节（用户输入时触发）
    close(done)                     // 关闭done，广播退出
}()

// 比如 goroutine 中可以调用此方法，检测是否被取消
func cancelled() bool {
    select {
    case <-done:
        return true  // 如果done已关闭，收到零值，返回true（退出）
    default:
        return false  // 否则，返回false（继续工作）
    }
}
```

> [Goroutine退出机制](https://juejin.cn/post/7124500225997144078)

# 并发竞争
## sync.Mutex 互斥锁
Go 提供了`sync.Mutex`：
```go
import "sync"

var (
    mu      sync.Mutex // guards balance
    balance int
)

func Deposit(amount int) {
    mu.Lock()
    balance = balance + amount
    mu.Unlock()
}

func Balance() int {
    mu.Lock()
    b := balance
    mu.Unlock()
    return b
}

// 实际建议使用 defer 来释放锁
func Balance() int {
    mu.Lock()
    defer mu.Unlock()
    return balance
}
```

> **注意：go里没有重入锁**

## sync.RWMutex读写锁
```go
var mu sync.RWMutex
var balance int
func Balance() int {
    mu.RLock() // readers lock
    defer mu.RUnlock()
    return balance
}
```

## sync.Once惰性初始化
初始化可能产生并发问题，可以使用`sync.Once`来确保初始化只执行一次。（这个场景类似 Java 的线程安全单例模式，或者Kotlin的object类）
```go
var loadIconsOnce sync.Once
var icons map[string]image.Image
// Concurrency-safe.
func Icon(name string) image.Image {
    loadIconsOnce.Do(loadIcons)
    return icons[name]
}
```

## sync.WaitGroup
`sync.WaitGroup` 是一个用于同步多个 Goroutine 的工具。它可以让主程序（或其他 Goroutine）等待一组 Goroutine 完成任务后再继续执行。

sync.WaitGroup 有三个主要方法：
* Add(n int): 增加计数器，表示有 n 个任务（Goroutine）需要等待完成。通常在启动 Goroutine 前调用。
* Done(): 减少计数器，表示一个任务完成。通常在 Goroutine 结束时调用。
* Wait(): 阻塞当前 Goroutine，直到计数器归零（所有任务完成）。

```go
func worker(id int, wg *sync.WaitGroup) {
	defer wg.Done() // 任务结束时调用 Done，计数器减 1
	fmt.Printf("Worker %d starting\n", id)
	time.Sleep(time.Second) // 模拟耗时工作
	fmt.Printf("Worker %d done\n", id)
}

func main() {
	var wg sync.WaitGroup // 创建 WaitGroup

	// 启动 3 个 Goroutine
	for i := 1; i <= 3; i++ {
		wg.Add(1) // 每个 Goroutine 增加计数器
		go worker(i, &wg) // 传递 WaitGroup 的指针
	}

	wg.Wait() // 等待所有 Goroutine 完成
	fmt.Println("All workers done")
}
```
注意需要确保 Add 在 go 语句之前调用，否则可能导致 Wait 提前返回（计数器已为 0）。

## 竞争条件检测工具
Go的runtime和工具链为我们提供了一个复杂但好用的动态分析工具，竞争检查器（the race detector）。竞争检查器会报告所有的已经发生的数据竞争。然而，它只能检测到运行时的竞争条件；并不能证明之后不会发生数据竞争。所以为了使结果尽量正确，请保证你的测试并发地覆盖到了你的包。

# Go开发环境
## 配置开发环境
例如 macOS 系统，使用 go 的 pkg 包安装后，安装程序会把 Go 的相关文件放到`/usr/local/go`路径，并且会在`/etc/paths.d/go`文件中写入 go 的 bin 目录。macOS 的 shell 启动时会自动读取 /etc/paths 和 /etc/paths.d/* 中的所有路径，并追加到 $PATH 中，无需用户手动配置。如果是下载文件自行放置到特定目录，就需要手动配置 PATH 环境变量。

## 项目结构
Go 的旧版本（1.11 之前）必须使用`GOPATH`环境变量，代码必须放在`GOPATH/src`目录中。
```
GOPATH/
├── src/                 # 源代码目录
│   └── github.com/
│       └── username/
│           └── project/
├── pkg/                 # 编译后的包
└── bin/                 # 编译后的可执行文件
```
Go 1.8 引入了 GOPATH 的默认值：`$HOME/go`

`GOROOT` 为 Go 的安装目录，默认值为 `/usr/local/go`。Go编译器会从`$GOPATH/src`和`$GOROOT/src`查找依赖包。开发者通过go get命令下载依赖到`$GOPATH/src`


但`GOPATH`存在明显的问题：
1. 单一工作空间，所有项目共享同一个GOPATH，导致项目隔离性差。
2. 依赖管理困难：没有版本控制，go get总是拉取最新版本，可能导致依赖不稳定。
3. 路径依赖：代码必须放在`$GOPATH/src`下，限制了项目目录的灵活性。


Go 1.11 版本引入了 Module，项目可以在任意目录下初始化模块（go mod init <module-name>），不再需要放在`$GOPATH/src`。依赖通过go get或go mod tidy管理，自动下载并记录在go.mod中。

基本的 Go 模块结构如下：
```
MyModulePath/
├── go.mod      # 模块定义文件，包含模块名、版本号等信息
├── xxx.go      # 源代码文件
├── ...         # 其他目录和源代码文件
```

使用模块后，`$GOPATH`仍然用于缓存模块（$GOPATH/pkg/mod）和存放二进制文件（$GOPATH/bin）。

> go env 可以查看当前 go 的涉及的所有环境变量的值

## Package
Go语言中，package的名称由文件顶部的package声明决定，而不是由文件所在的目录层级决定。在 Go 中，如果一个名字以大写字母开头，那么它就是已导出的。

两种方式都可以导入包：
```go
import "fmt"
import "os"

import (
    "fmt"
    "os"
)
```

如果包名相同，可以使用使用别名避免冲突：
```go
import (
    "crypto/rand"
    mrand "math/rand" // alternative name mrand avoids conflict
)
```

如果只是导入一个包而并不使用导入的包将会导致一个编译错误。但是有时候我们只是想利用导入包而产生的副作用：它会计算包级变量的初始化表达式和执行导入包的init初始化函数，使用匿名导入
```go
import _ "image/png"
```

`main`包比较特殊。它定义了一个独立可执行的程序，而不是一个库。在`main`包里的`main函数`也很特殊，它是整个程序执行时的入口。

## 命令工具
Go 语言的工具包含了包管理器、构建系统、测试的功能。

* go build xxx.go // 构建可执行文件保存在文件，只保留最后的可执行文件之外
* go run xxx.go // 直接编译运行 结合了构建和运行的两个步骤
* go install xxx.go // 构建可执行文件保存在文件，并放到 $GOPATH/bin 目录下（也就可以直接在命令行中运行），一般就是~/go/bin，每个包的编译结果也会保存

`gofmt`工具把代码格式化为标准格式（这个格式化工具没有任何可以调整代码格式的参数，只有固定的格式）， go 工具中的 fmt 子命令可以指定包，否则默认为当前目录中所有 go 源文件应用 gofmt 命令。

> Go语言的规范并没有指明包的导入路径字符串的具体含义，导入路径的具体含义是由构建工具来解释的。

`go get`命令支持当前流行的托管网站GitHub、Bitbucket和Launchpad，当前 Go Module 模式下，`go get`命令将获取特定版本的源代码文件，放在`$GOPATH/pkg/mod`目录内（`$GOPATH/pkg/mod/cache/download/`目录会放置原始的下载缓存压缩包和元数据）。 

> `go get`以源码依赖形式引入包，因为 Go 语言是native语言，不适合像Java那样依赖预编译的二进制文件。

需要注意的是导入路径含有的网站域名和本地Git仓库对应远程服务地址并不一定相同，这其实是Go语言工具的一个特性，可以让包用一个自定义的导入路径，但是真实的代码却是由更通用的服务提供，例如googlesource.com或github.com。因为页面 https://golang.org/x/net/html 包含了如下的元数据，它告诉Go语言的工具当前包真实的Git仓库托管地址：
```
$ go build gopl.io/ch1/fetch
$ ./fetch https://golang.org/x/net/html | grep go-import
<meta name="go-import"
      content="golang.org/x/net git https://go.googlesource.com/net">
```

# 反射
反射是由 reflect 包提供的。它定义了两个重要的类型：`Type`和`Value`。

例如这里，TypeOf 函数返回的是一个 Type 类型的值，ValueOf 函数返回的是一个 Value 类型的值。
```go
t := reflect.TypeOf(3)  // a reflect.Type
fmt.Println(t.String()) // "int"
fmt.Println(t)          // "int"
fmt.Printf("%T\n", 3) // "int"  使用 %T 打印类型

v := reflect.ValueOf(3) // a reflect.Value
fmt.Println(v)          // "3"
fmt.Printf("%v\n", v)   // "3"  使用 %v 打印值
fmt.Println(v.String()) // NOTE: "<int Value>"

t := v.Type()           // a reflect.Type ，通过 Value 可以获取到 Type
fmt.Println(t.String()) // "int"
```

reflect.ValueOf 的逆操作是 reflect.Value.Interface 方法。它返回一个 interface{} 类型：
```go
v := reflect.ValueOf(3)  // 创建一个 reflect.Value，包装了整数 3
x := v.Interface()       // 将 reflect.Value 转换回 interface{}，此时 x 包含值 3
i := x.(int)             // 类型断言：将 interface{} 转换为 int 类型
```

通过`reflect.Value`的`Kind()`方法，可以对类型分类：
```go
func format(v reflect.Value) string {
    switch v.Kind() {
    case reflect.Invalid:
        return "invalid"
    case reflect.Int, reflect.Int8, reflect.Int16,
        reflect.Int32, reflect.Int64:
        return strconv.FormatInt(v.Int(), 10)
    case reflect.Uint, reflect.Uint8, reflect.Uint16,
        reflect.Uint32, reflect.Uint64, reflect.Uintptr:
        return strconv.FormatUint(v.Uint(), 10)
    // ...floating-point and complex cases omitted for brevity...
    case reflect.Bool:
        return strconv.FormatBool(v.Bool())
    case reflect.String:
        return strconv.Quote(v.String())
    case reflect.Chan, reflect.Func, reflect.Ptr, reflect.Slice, reflect.Map:
        return v.Type().String() + " 0x" +
            strconv.FormatUint(uint64(v.Pointer()), 16)
    default: // reflect.Array, reflect.Struct, reflect.Interface
        return v.Type().String() + " value"
    }
}
```

##  通过reflect.Value修改值
先看看通过`reflect.ValueOf()`获取的内容：
```go
x := 2                   // value   type    variable?
a := reflect.ValueOf(2)  // 2       int     no
b := reflect.ValueOf(x)  // 2       int     no
c := reflect.ValueOf(&x) // &x      *int    no
d := c.Elem()            // 2       int     yes (x)
```
a只是整数2的拷贝，b只是x的值拷贝，c只是一个指针`&x`的拷贝，他们本身都不是真正的变量，因为所有通过`reflect.ValueOf(x)`返回的`reflect.Value`都是不可取地址的。但是对于d，它是c的解引用方式生成的，指向另一个变量，因此是可取地址的。我们可以通过调用reflect.`ValueOf(&x).Elem()`，来获取任意变量x对应的可取地址的Value。

通过调用reflect.Value的`CanAddr()`方法可以判断其是否可以被取地址：
```go
fmt.Println(a.CanAddr()) // "false"
fmt.Println(b.CanAddr()) // "false"
fmt.Println(c.CanAddr()) // "false"
fmt.Println(d.CanAddr()) // "true"
```

所以要通过`reflect.Value`修改值，需要先调用`Addr()`方法得到包含指向变量的指针的Value，然后得到对应的`interface{}`，最后通过类型断言将得到的`interface{}`类型的接口强制转为普通的类型指针，通过这个指针就可以修改值了：
```go
x := 2
d := reflect.ValueOf(&x).Elem()   // d refers to the variable x
px := d.Addr().Interface().(*int) // px := &x
*px = 3                           // x = 3
fmt.Println(x)                    // "3"
```

或者直接使用可取地址的`reflect.Value`值的`Set`方法来更新对应的值：
```go
d.Set(reflect.ValueOf(4))
fmt.Println(x) // "4"
```
还有一些用于基本数据类型的Set方法：SetInt、SetUint、SetString和SetFloat

对于一个引用interface{}类型的reflect.Value调用SetInt会导致panic异常，即使那个`interface{}`变量对应整数类型也不行。
```go
x := 1
rx := reflect.ValueOf(&x).Elem()
rx.SetInt(2)                     // OK, x = 2
rx.Set(reflect.ValueOf(3))       // OK, x = 3
rx.SetString("hello")            // panic: string is not assignable to int
rx.Set(reflect.ValueOf("hello")) // panic: string is not assignable to int

var y interface{}
ry := reflect.ValueOf(&y).Elem()
ry.SetInt(2)                     // panic: SetInt called on interface Value
ry.Set(reflect.ValueOf(3))       // OK, y = int(3)
ry.SetString("hello")            // panic: SetString called on interface Value
ry.Set(reflect.ValueOf("hello")) // OK, y = "hello"
```

反射可以越过Go语言的导出规则的限制读取结构体中未导出的成员，比如在类Unix系统上os.File结构体中的fd int成员。然而，利用反射机制并不能修改这些未导出的成员。`CanSet`是用于检查对应的`reflect.Value`是否是可取地址并可被修改的：
```go
fmt.Println(fd.CanAddr(), fd.CanSet()) // "true false"
```

## reflect.Method
reflect.Type和reflect.Value 都可以调用 `Method(int)`返回一个`reflect.Method`实例。`reflect.Value.Call`可以调用方法。

# 底层编程
## unsafe包
unsafe包是一个采用特殊方式实现的包，是由编译器实现的，它提供了一些访问语言内部特性的方法，特别是内存布局相关的细节。使用`unsafe`包来摆脱Go语言规则带来的限制，例如使用`unsafe.Sizeof`、`unsafe.Alignof`、`unsafe.Offsetof`和`unsafe.Pointer`

## 使用cgo调用C代码
