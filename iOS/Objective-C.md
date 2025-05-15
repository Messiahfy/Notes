https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC
https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide

Objective-C 是 C 的超集，所以 C 语言的语法在 Objective-C 中仍然适用，Objective-C 特有的语法主要在面向对象这方面。

Objective-C 为 ANSI C 添加了下述语法和功能：
* 定义新的类
* 类和实例方法
* 方法调用（称为发消息）
* 属性声明（以及通过它们自动合成存取方法）
* 静态和动态类型化
* 块 (block)，已封装的、可在任何时候执行的多段代码
* 基本语言的扩展，例如协议和类别

## 类的定义
在OC中，定义类分为声明部分和实现部分，一般分别放在`.h`和`.m`文件中。

1. 声明部分：
```
// SimpleClass.h

#import <Foundation/Foundation.h>

@interface SampleClass : NSObject

// 属性
@property int age;
@property NSString *name;

// 返回 NSString 指针的无参方法
- (NSString *)testMethod0;

// 返回int类型的1个参数的方法
- (int)testMethod1:(int)a;

// 返回int类型的2个参数的方法
- (int)testMethod2:(int)a second:(Boolean)b;

// 类方法（静态方法）
+ (void)testClassMethod;

@end
```
`@interface`关键字用于定义一个类的声明部分，`@interface`和`@end`之间的内容是类的公开接口部分，包括属性和方法的声明。

* 减号（-）表示实例方法，加号（+）表示类方法。
* @property用于声明属性，下文详细分析。
* OC的方法或者说函数的声明和使用和常用的语言的语法不一样，不过看一下也就了解了。
* 类必须直接或间接继承自NSObject类。

2. 实现部分：
```
// SimpleClass.m

#import "SimpleClass.h"

@implementation SampleClass

- (NSString *)testMethod0 {
    return @"测试无参方法";
}

- (int)testMethod1:(int)a {
    return a + 2;
}

- (int)testMethod2:(int)a second:(Boolean)b {
    if (b) {
        return a + 3;
    }
    return a;
}

+ (void)testClassMethod {
    NSLog(@"测试类方法");
}

@end
```
`@implementation`关键字用于定义一个类的实现部分，`@implementation`和`@end`之间的内容是类的实现部分，包括方法定义。

> 实际上，声明和实现也可以写到一个文件里面，无论是`.h`还是`.m`，但是必须保证这个文件只被`import`一次，因为这个文件包含了实现部分，这和C语言是一样的，声明可以重复，但是定义（实现）不能重复，重复引入会产生duplicate symbols错误。

> 类的实例方法和普通函数不一样，普通函数是C风格，而实例和类方法是Objective-C风格的。

## 使用对象
OC调用方法，称为发送消息，使用方括号
```
// 调用类方法
[SampleClass testClassMethod];

// 创建对象并调用实例方法
SampleClass *instance = [[SampleClass alloc] init];
NSLog(@"测试初始的属性：%d %@", instance.age, instance.name);
instance.age = 19;
instance.name = @"哈哈";
NSLog(@"测试修改后的属性：%d %@", instance.age, instance.name);
NSString *result = [instance testMethod0];
NSLog(@"测试无参方法：%@", result);

int a = [instance testMethod1:1];
NSLog(@"测试1个参数方法：%d", a);

int b = [instance testMethod2:1 second:true];
NSLog(@"测试2个参数方法：%d", b);
```

## 类别 Category
类别可以用于为已存在的类加入新的方法（不能添加属性）。类别的语法如下：
```
@interface ClassName (CategoryName)
// ......
@end

@implementation ClassName (CategoryName)
// ......
@end
```

假设已存在一个类MyPoint：
```
// MyPoint.h
@interface MyPoint : NSObject

@property int x;
@property int y;

@end

// MyPoint.m
@implementation MyPoint
@end
```

现在我想为MyPoint类添加一个打印方法，不能修改MyPoint源码的情况下，可以使用类别这样做：
```
// MyPointCategory.h
#import "Point.h"

@interface MyPoint (MyPointPrint)

-(void)print;

@end


// MyPointCategory.m
#import <Foundation/Foundation.h>
#import "PointCategory.h"

@implementation MyPoint (MyPointPrint)

-(void)print {
    NSLog(@"x：%d  y: %d", self.x, self.y);
}

@end
```
通过类别，为MyPoint添加了print方法。


除了可以为已有类添加新的方法外，你还`可以用类别把那些实现起来很复杂的类分解成多个源代码文件`。或者，你`也可以根据你软件所在的平台（macOS或iOS），来对类别方法提供不同的实现`。实例方法和类方法都可以在类别中声明，但是一般不在类别中声明一个新的属性。尽管你能在类别中声明一个额外属性，但你却不能声明一个相应的实例变量。这意味着编译器既不会自动生成类别中新增加的属性对应的实例变量，也不会自动生成这些属性的存取方法。你可以在分类中实现自己的存取方法，但你写的存取方法只能读取原来类保存的属性。

为一个已有类添加新的属性的唯一方法就是使用类的扩展(class extension),

> 如果一个定义在类别中的方法名和原来的类中的一个方法名重复，或者和该类的另一个类别的一个方法名重复了，在运行过程中具体调用哪个方法来实现会是随机决定的。为了避免这些不必要的冲突，很有必要为新添加的方法名添加一个前缀。


## 扩展 Extension
类的拓展和类别有点相似，但类的拓展只能在你有这个类的实现的源代码时使用，把扩展直接写到类的实现部分所在的`.m`文件中。类的拓展可以添加它的属性和实例变量，编译器会自动实现拓展中出现的属性和实例变量的get方法和set方法。

`MyPoint`类的声明部分：
```
#import <Foundation/Foundation.h>

@interface MyPoint : NSObject

@property int x;
@property int y;

- (void)print;

@end
```

类的扩展写到实现部分的文件中：
```
#import "Point.h"
#import <Foundation/Foundation.h>

// MyPoint类的扩展，增加了一个私有属性z和私有方法privatePrint
@interface MyPoint ()

@property int z;
- (void)privatePrint;

@end

// MyPoint类的实现部分
@implementation MyPoint

- (void)print {
    self.z = 30;
    [self privatePrint];
}

- (void)privatePrint {
    NSLog(@"x : %d   y : %d  z : %d", self.x, self.y, self.z);
}

@end
```
**扩展中增加的属性和方法都是私有的**，所以只能在类的内部使用，外部代码无法直接访问（除非通过运行时反射）。


**拓展中也可以增加新的实例变量**，新增添的实例变量须在拓展中用花括号括起来：
```
@interface MyPoint () {

    id _someCustomInstanceVariable;

}

...

@end
```

> 扩展适用于隐藏私有属性和方法，分离公共和私有接口。举个例子，在公有部分定义一个readonly的公有属性，再在类的拓展中定义一个readwrite的私有属性，这样，外部就可以访问到一个类内部经常变化的一个属性，却不会去改变它。

## 属性
在OC中声明属性有两种方式，一种是使用`@property`关键字，一种是直接声明实例变量。

### 直接声明实例变量（不推荐）
这是一种不太推荐使用的方式，但可以了解一下。不使用`@property`声明的实例变量可以把它们放在一对花括号中，这对花括号需要位于类声明部分或实现部分顶部，像这样：
```
@interface SomeClass : NSObject {
    NSString *_myNonPropertyInstanceVariable;
}
...
@end

@implementation SomeClass {
    NSString *_anotherCustomInstanceVariable;
}
...
@end
```

默认情况下，实例变量的访问权限是`protected`，我们也可以手动指定访问权限：
```
#import <Foundation/Foundation.h>

@interface SampleClass : NSObject {

    int defaultItem; // 默认就是 @protected

  @public
    int publicItem;

  @protected
    int protectedItem;

  @private
    int privateItem;
}

// 这里还是上面介绍声明类的时候声明的属性和方法
@property int age;

- (NSString *)testMethod0;

@end

@interface SubSampleClass : SampleClass

// 子类可以访问父类的 protected 和 public 实例变量，但不能访问 private 实例变量
-(void)test{
    self->protectedItem;
    self->publicItem;
    self->defaultItem;
}
@end
```

```
// 外部代码，只能访问 public 实例变量
SampleClass *instance = [[SampleClass alloc] init];
instance->publicItem;
```

1. 实例变量可以在实现中访问，所以也就可以手动封装 getter/setter 方法，暴露给外部使用
2. @protected也可以在子类访问
3. 实例变量需要通过 `->` 操作符访问
4. 如果是在 @implementation 中声明的实例变量，只能在当前文件中访问，外部代码无法通过 `->` 访问
5. 使用运行时反射访问，则可以忽略访问权限

### 使用@property关键字
`@property` 是 Objective-C 中的一个编译器指令（directive），用于简化类的属性声明和实现。它主要用于自动生成 getter 和 setter 方法，以及管理实例变量的内存和访问。

以这个简单的例子来了解`@property`：
```
@interface Person : NSObject

@property NSString *name;

@end
```

#### 1. 自动生成 getter 和 setter 方法

* Getter: - (NSString *)name;
* Setter: - (void)setName:(NSString *)name;。

生成的伪代码如下：
```
@interface Person : NSObject {
    NSString *_name; // 自动生成实例变量
}
- (NSString *)name; // Getter
- (void)setName:(NSString *)name; // Setter
@end

@implementation Person
- (NSString *)name {
    return _name;
}
- (void)setName:(NSString *)name {
    _name = name; // 默认 assign 语义
}
@end
```

所以在使用时，可以直接调用getter和setter方法：
```
Person *instance = [[Person alloc] init];

// 调用 name 方法
NSString *name = [instance name];
// 调用 setName 方法
[instance setName:@"哈哈"];
```

**并且使用`@property`声明的属性，可以使用点语法更便捷地调用getter和setter方法：**
```
NSString *name = instance.name;
instance.name = @"哈哈";
```

> 点语法依赖的是getter和setter方法的存在，并不要求一定是`@property`自动生成的方法，所以也可以直接声明实例变量并且手动实现getter和setter方法，也可以使用点语法。

#### 2. 关联实例变量
`@property`声明的属性会自动关联一个实例变量，默认以 `_` 开头，上面的伪代码展示了自动生成的`_name`实例变量。

使用 `@synthesize` 可以自定义属性生成的实例变量的名称，比如不想叫 `_name`，可以这样写：
```
@interface Person : NSObject
@property NSString *name;
@end

@implementation Person
@synthesize name = customName; // 将 name 属性关联到 customName 实例变量
@end
```

如果使用了`@synthesize`但没有为其规定一个实例变量名，像这样：
```
@synthesize name;
```
实例变量的名字将会和属性相同为`name`，没有下划线。

### 属性修饰符
`@property` 声明属性时可以通过属性修饰符（attributes）来控制属性的行为，包括内存管理、线程安全、读写权限以及 getter/setter 方法的命名等。

#### 读写修饰符
`readwrite`是默认的读写修饰符，会让属性生成getter和setter。如果想让属性只读，可以使用`readonly`，这样就只会生成getter方法，但一般需要重写getter方法，不然获取到的属性值总是nil（对象类型的情况），因为无法赋值。

例如这里就重写了fullName的getter方法，返回一个拼接好的字符串。
```
// 声明部分（省略其他代码）
@property (readonly) NSString *fullName;


// 实现部分（省略其他代码）
- (NSString *)fullName {
    return [NSString stringWithFormat:@"%@ %@", self.firstName, self.lastName];
}
```
或者在类扩展中重新声明为 `readwrite`，实现内部可读写，外部只读。

### 原子性修饰符
默认使用`atomic`，表示属性的单次读写是原子的，自动生成的getter和setter会添加锁，也可以设置为`nonatomic`。

```
@property (nonatomic) NSString *name;
```
iOS 开发中常用`nonatomic`，因为 UIKit 通常在主线程操作，使用 `nonatomic` 可以提高性能。

### 内存管理修饰符

* 强引用
```
@property (strong) NSString *name;
```
对象类型的 `@property` 默认使用 `strong`

* 弱引用
```
@property (weak) id<SomeDelegate> delegate;
```

* copy
```
// copy，在 setter 中创建对象的副本（通过调用 [value copy]），而不是直接引用传入的对象
@property (copy) NSString *name;

// copy对应的setter伪代码
- (void)setName:(NSString *)name {
    _name = [name copy]; // 创建副本
}
```

* assign 和 unsafe_unretained

基本数据类型的 `@property` 默认使用 `assign`。
```
@property (assign) int age;
```
`assign`表示属性的 setter 方法直接赋值，不执行任何内存管理操作（不增加引用计数，也不释放旧值），对于基本类型和结构体，自然如此。在 MRC情况下，`@property`默认会给对象类型的属性生成的setter方法添加上retain、release方法用于引用计数管理，如果使用`assign`则不会自动添加引用计数管理。在 ARC 情况下，一般不对对象类型使用这个修饰符。

`unsafe_unretained`也是给对象类型使用，作用和`assign`使用在对象类型上时类似，它们都和`weak`一样不增加引用计数，但区别是它们引用的对象被释放的时候，这个指针变量不会被设置为nil。这也就意味着这个指针会指向一段被释放的内存，我们称其为 野指针（dangling pointer），这就是为什么它被称作为“不安全”的。向一个野指针发送消息会导致崩溃。

* retain：和 strong 类似，但仅在 MRC 下使用。。

### 自定义方法名修饰符
```
@property (getter=isValid) BOOL valid;

@property (setter=updateName:) NSString *name; // 自定义 setter 方法必须符合 Objective-C 方法签名（带一个参数）
```

大多数情况我们还是使用点语法访问属性，并不受自定义方法名的影响，除非使用调用方法的方式访属性。
```
[instance updateName:  @"名称"];
instance.name = @"名称";
```

### copy
在属性声明时使用`copy`，自动生成 setter 方法时会创建对象的副本
```
@property (copy) NSString *name;

// 自动生成的 setter 伪代码
- (void)setName:(NSString *)name {
    _name = [name copy]; // 创建不可变副本
}
```

所有你希望设置为`copy`的属性类型必须遵从`NSCopying`协议，`NSString`等系统类型已经实现。
```
// MyClass.h
@interface MyClass : NSObject <NSCopying>
@property (strong) NSString *content;
@end

// MyClass.m
@implementation MyClass

- (id)copyWithZone:(NSZone *)zone {
    MyClass *copy = [[[self class] alloc] init]; // 使用标准alloc
    copy.content = [self.content copy]; // 深拷贝属性
    return copy;
}

@end
```

### @dynamic
`@dynamic` 是一个属性声明关键字，用于告诉编译器：属性的 getter 和 setter 方法不会在编译时自动生成，而是由开发者在运行时动态提供。

比如可以使用 Objective-C 运行时的 resolveInstanceMethod: 或 resolveClassMethod: 方法，在运行时动态添加方法实现。

## 初始化方法
了解初始化方法之前，先看一下基类`NSObject`中的相关方法：
### NSObject
先了解基类NSObject中的一些方法

1. 类方法：
* `load`: 在类或类别（category）被加载到运行时时由OC Runtime自动调用。用于类或类别加载时的初始化，比 initialize 更早。
* `initialize`：当类第一次被使用时由OC Runtime调用，用于初始化类级别的静态数据（例如共享资源、配置）。可以在实现中继承重写。
* `alloc`：为对象分配内存，返回未初始化的对象，是对象创建的第一步，通常与`init`配对使用。不要直接使用未初始化的对象（可能导致未定义行为）。
* new：一步创建并初始化对象，等价于`alloc`和`init`的组合。但不支持自定义 init 方法（如 initWith...）
* 

2. 实例方法：
* `init`：初始化方法，通常在`alloc`之后调用，用于对象的初始化。
* `copy`：创建对象的副本（深拷贝或浅拷贝，取决于实现）。
* `dealloc`：在对象释放（内存回收）时调用。自动引用计数（ARC）：对象引用计数为 0 时，运行时自动调用；手动引用计数（MRC）：开发者调用 [object release] 触发。

### 自定义初始化方法
Objective-C 的初始化方法遵循以下约定：
1. 方法命名：
  * 初始化方法以 `init` 开头，例如 `init`, `initWithName:`, `initWithFrame:`。
  * 返回类型通常是 id（动态类型）或类的具体类型（例如 MyClass *）。
2. 初始化流程：
  * 调用父类的初始化方法（通常是 `[super init]` 或其变体）以确保父类部分被正确初始化。
  * 检查父类初始化是否成功（返回值不为 `nil`）。
  * 初始化子类的实例变量或其他状态。
  * 返回初始化后的对象（通常是 `self`）。

oc的初始化方法一般实现如下：
```
// 现代 Objective-C 推荐使用 instancetype 作为初始化方法的返回类型，而不是 id
- (instancetype)init {
    self = [super init]; // 调用父类的初始化方法
    if (self) {          // 检查父类初始化是否成功
        // 初始化子类的实例变量
        _someProperty = @"DefaultValue";
    }
    return self;         // 返回初始化后的对象
}
```

例子：
```
@interface Person : NSObject
@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSInteger age;
- (instancetype)initWithName:(NSString *)name age:(NSInteger)age;
@end

@implementation Person
- (instancetype)initWithName:(NSString *)name age:(NSInteger)age {
    self = [super init]; // 调用父类的指定初始化方法
    if (self) {
        _name = [name copy]; // 初始化 name 属性
        _age = age;          // 初始化 age 属性
    }
    return self;
}
@end

// 使用
Person *person = [[Person alloc] initWithName:@"John" age:30];
```

### 初始化方法的继承
**在 Objective-C 中，子类会继承父类的所有初始化方法（init 方法）**，所以虽然`Person`类没有定义自己的`init`方法，仍然可以调用继承父类的`init`方法：
```
Person *p = [[Person alloc] init];
NSLog(@"name age %@  %ld", p.name, (long)p.age);

// 打印结果：
name age (null)  0
```
因为继承父类的`init`方法并没有初始化`Person`自身的属性，所以打印的属性值都是默认值。

我们可以覆盖父类的`init`方法，来初始化`Person`的属性：
```
- (instancetype)initWithName:(NSString *)name {
    return [self initWithName:name age:0]; // 调用指定初始化方法，提供默认 age
}

- (instancetype)init {
    return [self initWithName:@"Unknown" age:0]; // 调用指定初始化方法，提供默认值
}
```
**虽然子类会继承父类的所有初始化方法，但父类的初始化方法可能无法正确初始化子类的特有属性或状态，所以要么覆盖父类的初始化方法，添加子类特有的初始化逻辑，要么子类定义新的初始化方法，并在必要时禁用父类的初始化方法（子类可以覆盖这些方法并抛出异常）**

## 文件扩展名

|扩展名|源类型|
| --- | --- |
| .h | 头文件。头文件包含类、类型、函数和常量声明。 |
| .m | 实现文件。具有此扩展名的文件可以同时包含 Objective-C 代码和 C 代码。有时也称为源文件。 |
|.mm | 实现文件。具有此扩展名的实现文件，除了包含 Objective-C 代码和 C 代码以外，还可以包含 C++ 代码。仅当您实际引用您的 Objective-C 代码中的 C++ 类或功能时，才使用此扩展名。 |

## 协议
协议可以包括实例方法、类方法以及属性的声明，但不能定义实例变量。以下示例展示使用一个协议声明和使用：
```
@protocol MyProtocol
- (void)myProtocolMethod;
@end

@interface MyClass : NSObject <MyProtocol>
...
@end
```

声明遵循某种协议的属性：
```
// 类中的属性
@interface MyView : UIView
@property (weak) id <MyProtocol> dataSource;
...
@end

// 普通变量
id <MyProtocol> processor = [self xxxx];
```


采用多个协议的情况：
```
@interface MyClass : NSObject <MyProtocol, AnotherProtocol, YetAnotherProtocol>
...
@end
```

### 协议可选方法
我们可以使用`@optional`指令将协议方法标记为可选方法，`@optional`指令适用于紧随其后的所有方法，直到协议的定义结束或者遇到另一个指令，例如`@required`。
```
@protocol MyProtocol
- (NSUInteger)method1;
- (CGFloat)method2:(NSUInteger)a;
@optional
- (NSString *)method3:(NSUInteger)a;
- (BOOL)method4:(NSUInteger)a;
@required
- (BOOL)method5:(NSUInteger)a;
@end
```
该示例定义了一个具有三个必需方法和两个可选方法（这两个可选方法处于指令`@optional`和`@required`之间）的协议。

### 检查可选方法是否在运行时实现
检查方式：
```
if ([object respondsToSelector:@selector(optionalMethod)]) {
    // 方法已实现
    [object optionalMethod];
} else {
    // 方法未实现
}
```

## 代码块
Blocks基本语法
```
^{
    NSLog(@"This is a block");
}

// 声明一个 simpleBlock 变量，并赋值
void (^simpleBlock)(void) = ^{
    NSLog(@"This is a block");
};

// 声明并实现了一个有参数和返回值的Block，可以像调用函数一样调用它：
double (^multiplyTwoValues)(double, double) = ^(double firstValue, double secondValue) {
  return firstValue * secondValue;
};
double result = multiplyTwoValues(2, 4);
```

在属性中使用
```
@interface MyObject : NSObject
@property (copy) void (^blockProperty)(void);
@end
```

### Blocks可以捕获封闭作用域内的值
```
int multiplier = 7;
int (^myBlock)(int) = ^(int num) { return num * multiplier; };

int result = myBlock(4); // result is 28
```

如果您实现一个方法，并且该方法定义一个块，则该块可以访问该方法的局部变量和参数（包括堆栈变量），以及函数和全局变量（包括实例变量）。这种访问是只读的，或者说捕获的值是快照/拷贝。但如果使用 `__block` 修饰符声明变量，则可在块内更改其值。即使包含有块的方法或函数已返回，并且其局部作用范围已销毁，但是只要存在对该块的引用，局部变量仍作为块对象的一部分继续存在。
```
__block int anInteger = 42;

void (^testBlock)(void) = ^{
    NSLog(@"Integer is: %i", anInteger);
    anInteger = 100;
};

testBlock();
NSLog(@"Value of original variable is now: %i", anInteger);

// 打印结果：
Integer is: 42
Value of original variable is now: 100
```

### Blocks可以作为参数和返回值
```
// 声明
- (void)beginTaskWithCallbackBlock:(void (^)(void))callbackBlock;

// 实现
- (void)beginTaskWithCallbackBlock:(void (^)(void))callbackBlock {
    ...
    callbackBlock();
}


// 带参数的block的情况
- (void)doSomethingWithBlock:(void (^)(double, double))block {
    ...
    block(21.0, 2.0);
}
```

### 用类型定义（Type Definition）
```
// 先定义一个block类型
typedef void (^SimpleBlock)(void);

// 之后使用更加易读
SimpleBlock anotherBlock = ^{
    ...
};

- (void)beginFetchWithCallbackBlock:(SimpleBlock)callbackBlock {
    ...
    callbackBlock();
}
```

### 注意Block循环引用
```
- (void)configureBlock {
    MyBlockKeeper * __weak weakSelf = self;
    self.block = ^{
        [weakSelf doSomething];   // 捕获了弱引用
                                  // 避免了引用循环
    }
}
```

## 错误处理
在 Objective-C 中，NSError 用于处理可预期的错误，而异常处理（@try/@catch）用于处理不可预期的程序错误。

### 使用 NSError 处理不可避免错误
```
// 定义一个可能失败的方法
- (BOOL)doSomethingWithError:(NSError **)error {
    if (/* 成功条件 */) {
        return YES;
    } else {
        // 创建错误对象
        if (error) {
            *error = [NSError errorWithDomain:@"com.yourdomain.app"
                                        code:1001
                                    userInfo:@{
                NSLocalizedDescriptionKey: @"操作失败描述",
                NSLocalizedFailureReasonErrorKey: @"失败原因说明"
            }];
        }
        return NO;
    }
}

// 调用示例
NSError *error = nil;
if (![self doSomethingWithError:&error]) {
    NSLog(@"错误: %@", error.localizedDescription);
    // 处理错误
}
```
这里使用了双重指针，因为如果使用普通指针（NSError *）就只能在方法内部修改指针指向的对象内容，而无法修改指针本身，也就不能让调用者的指针指向新的`NSError`对象。

### 异常处理
异常处理用来处理程序员的错误，比如越界访问、空指针引用等。
```
@try {
    // 可能抛出异常的代码
    NSArray *array = @[@"1", @"2"];
    NSString *item = array[3]; // 越界访问
}
@catch (NSException *exception) {
    NSLog(@"异常名称: %@", exception.name);
    NSLog(@"异常原因: %@", exception.reason);
}
@finally {
    // 无论是否发生异常都会执行的代码
    NSLog(@"清理工作");
}
```

## 内存管理
Objective-C使用引用计数管理内存，引用计数为0则释放对象内存。

1. MRC（手动引用计数）
OC的基类NSObject包含以下方法用于内存管理
* alloc/new/copy/mutableCopy：生成并持有对象（引用计数+1）
* retain：持有对象（引用计数+1）
* release：释放对象（引用计数-1）
* dealloc：废弃对象

2. ARC（自动引用计数）
编译器在适当的位置插入释放内存的代码

* __strong：强指针（默认），指针为nil或作用域结束就自动释放
* __weak：弱指针，不影响引用计数，没有强指针引用时，__weak指针引用变为nil。主要是解决循环引用等内存泄漏问题
* __unsafe_unretained：也不影响引用计数，但一个对象没有强指针引用时，__unsafe_unretained指针引用不会为nil，而是野指针
* __autoreleasing：

> MRC情况下，函数返回对象，要给调用方用，所以函数内不能释放，所以要么使用自动释放池，要么由调用方release（比如new、alloc开头的函数）

> **ARC情况下，函数返回对象，由于编译器会检查方法名是否以alloc/new/copy/mutableCopy开头，如果不是（例如`[NSString stringWithFormat:]`、`[NSArray array]`等便利方法）则自动将返回值的对象注册到autoreleasepool。**

> Cocoa/Cocoa Touch 框架会在主线程为每个 RunLoop 自动创建一个默认的 autorelease pool，并在 RunLoop 迭代结束时清空。如果是在例如命令行程序中使用Objective-C，那么需要手动创建一个 autorelease pool，否则可能内存泄露，始终在 main 函数中使用 @autoreleasepool，以确保 Foundation 框架的 autorelease 对象被正确释放。这是一个良好的编程习惯，即使你的代码当前不明显依赖 autorelease 对象，也能防止未来引入此类 API 时出现问题。

[iOS里的内存管理](https://www.jianshu.com/p/c3344193ce02)

苹果给出了三种需要手动添加@autoreleasepool 的情況：
1. 程序不是基于UI框架，比如普通命令行
2. 如果你编写的循环中创建了大量的临时对象；你可以在循环内使用 @autoreleasepool 在下一次迭代之前处理这些对象。在循环中使用 @autoreleasepool 有助于减少应用程序的最大内存占用。
3. 如果你创建了其他线程，一旦线程开始执行，就必须创建自己的 @autoreleasepool，否则，你的应用程序将存在内存泄漏。

例如在Objective-C中，使用`[NSString stringWithFormat:]`，由于ARC对于这种函数返回的对象不会在引用计数为0时立即释放内存，而是添加到当前的`autoreleasepool`中，当`autoreleasepool`作用域结束会自动释放这些内存。所以如下代码使用 `autoreleasepool` 的目的就是可以让 str 在当前循环内释放，否则会等到更外层的`autoreleasepool`作用域（使用cocoa touch时，UIApplicationMain函数会在每次runloop时创建一个新的 autorelease pool）结束才释放，导致内存占用过高。
```
for (int i = 0; i < 1000; i++) {
    @autoreleasepool {
        NSString *str = [[NSString alloc] initWithFormat:@"Object %d", i];
    } // autoreleasepool作用域结束，释放 str 对象
    // ... 当前循环其他代码，此时 str 对象已经释放
}
```
> 假如这里使用的是init/new/copy/mutableCopy开头的方法来创建str，那么就不需要手动添加@autoreleasepool，因为这种情况ARC会在str引用计数为0时（也就是本次循环的作用域结束时）立即释放。

[Objective-C Automatic Reference Counting (ARC)](https://clang.llvm.org/docs/AutomaticReferenceCounting.html)

## 通信方式
Protocol
Block
NSNotification
KVC（key-value coding）读写属性、KVO（key-value observing）监听属性


Runtime：https://www.ianisme.com/ios/2019.html
Runloop：https://blog.ibireme.com/2015/05/18/runloop/