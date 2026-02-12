`TypeScript`是`JavaScript`的超集，所有合法的`JavaScript`代码都是合法的`TypeScript`代码。这使得 `JavaScript` 项目可以逐步迁移到 `TypeScript`，而无需完全重写。

`TypeScript` 编译器，我们需要通过 npm 获取它。ts文件经过tsc可以转换为js文件。
```shell
npm install -g typescript
```

## 基本语法
变量声明
```ts
// 建议使用let和const来声明变量和常量
let a: number = 10;
const b: string = "hello";
```

### 函数
```ts
// 普通函数，参数和返回值都有类型（而js没有）
function sum(a: number, b: number): number {
  return a + b;
}

// 箭头函数
function greeter(fn: (a: string) => void) {
  fn("Hello, World");
}
 
function printToConsole(s: string) {
  console.log(s);
}
 
greeter(printToConsole);
```

#### 可选参数和默认参数
```ts
// 可选参数
function buildName(firstName: string, lastName?: string) {
    if (lastName)
        return firstName + " " + lastName;
    else
        return firstName;
}

// 默认参数
function buildName(firstName: string, lastName = "Smith") {
    return firstName + " " + lastName;
}
```

> 而JavaScript中，调用函数时传入的参数类型、个数都是很随便的。

#### 剩余参数
也就是其他语言中的可变参数，传递任意数量的参数：
```ts
function buildName(firstName: string, ...restOfName: string[]) {
  return firstName + " " + restOfName.join(" ");
}
```

## 面向对象
### 类
声明一个类：
```ts
class Greeter {
    // 属性和方法不需要加let、function关键字
    greeting: string;
    constructor(message: string) {
        this.greeting = message;
    }
    greet() {
        return "Hello, " + this.greeting;
    }
}

let greeter = new Greeter("world");
```

### 继承
```ts
class Animal {
    move(distanceInMeters: number = 0) {
        console.log(`Animal moved ${distanceInMeters}m.`);
    }
}

class Dog extends Animal {
    bark() {
        console.log('Woof! Woof!');
    }
}

const dog = new Dog();
dog.bark();
dog.move(10);
```

### 访问修饰符
默认为`public`，还可以使用`private`、`protected`修饰符

### readonly修饰符
只读属性必须在声明时或构造函数里被初始化。
```ts
class Greeter {
  readonly name: string = "world";
 
  constructor(otherName?: string) {
    if (otherName !== undefined) {
      this.name = otherName;
    }
  }
}
```

### 参数属性
参数属性可以直接在构造函数中声明，这样可以把声明和赋值合并至一处：
```ts
class Octopus {
    readonly numberOfLegs: number = 8;
    constructor(readonly name: string) {
    }
}
```

### getter/setter
```ts
class Employee {
    private _fullName: string;

    get fullName(): string {
        return this._fullName;
    }

    set fullName(newName: string) {
        if (passcode && passcode == "secret passcode") {
            this._fullName = newName;
        }
        else {
            console.log("Error: Unauthorized update of employee!");
        }
    }
}
```
> 只带有`get`不带有`set`会自动被推断为`readonly`

### 静态属性和静态方法
添加`static`关键字即可

### abstract
使用`abstract`可以声明抽象类和抽象方法

## 类型兼容性
TypeScript 的类型系统是结构性的 ，而不是名义上的：我们可以将 obj 用作 Pointlike，因为它具有 x 和 y 属性，它们都是数字。类型之间的关系取决于它们包含的属性，而不是它们是否使用某种特定关系声明。
```ts
interface Pointlike {
  x: number;
  y: number;
}
interface Named {
  name: string;
}
 
function logPoint(point: Pointlike) {
  console.log("x = " + point.x + ", y = " + point.y);
}
 
function logName(x: Named) {
  console.log("Hello, " + x.name);
}
 
const obj = {
  x: 0,
  y: 0,
  name: "Origin",
};
 
logPoint(obj);
logName(obj);
```


```ts
interface Named {
    name: string;
}

class Person {
    name: string;
}

let p: Named;
// OK, because of structural typing
p = new Person();
```


```ts
let x = (a: number) => 0;
let y = (b: number, s: string) => 0;

y = x; // OK
x = y; // Error
```


比较两个类类型的对象时，只有实例的成员会被比较。 静态成员和构造函数不在比较的范围内。

ts的类型，与Java、C#等语言有所不同，在 TypeScript 中，最好将类型视为一组共享某些共同点的值 。因为类型只是集合，所以特定值可以同时属于多个集合。


## 高级类型
### 联合类型
联合类型（Union Types）可以通过`|`将变量设置为多种类型，赋值时可以根据设置的类型来赋值。
```ts
let val:string|number 
val = 12 
console.log("数字为 "+ val) 
val = "Runoob" 
console.log("字符串为 " + val)
```

### 交叉类型
例如，`Person` & `Serializable` & `Loggable`同时是 `Person` 和 `Serializable` 和 `Loggable`。 这个类型的对象同时拥有了这三种类型的成员

### typeof 和 instanceof
typeof 用于检查基本类型，返回字符串形式，比如 string、number、boolean、symbol、undefined、function、object
```ts
function padLeft(value: string, padding: string | number) {
    if (typeof padding === "number") {
        return Array(padding + 1).join(" ") + value;
    }
    if (typeof padding === "string") {
        return padding + value;
    }
    throw new Error(`Expected string or number, got '${padding}'.`);
}
```

具体的对象类型，应该使用`instanceof`，用法和Java类似
```ts
let padder: Padder = getRandomPadder();

if (padder instanceof SpaceRepeatingPadder) {
    padder; // 类型细化为'SpaceRepeatingPadder'
}
if (padder instanceof StringPadder) {
    padder; // 类型细化为'StringPadder'
}
```

### 类型别名
可以使用**类型别名**来命名各种类型：
```ts
// 命名函数类型
type GreetingFunction = (a: string) => void;
function greeter(fn: GreetFunction) {
  // ...
}

// 命名字符串类型
type Name = string;

type NameResolver = () => string;
// 命名联合类型
type NameOrResolver = Name | NameResolver;
```

## 接口
只要一个对象包含了接口中定义的所有属性，它就被视为实现了该接口。
```ts
interface LabelledValue {
  label: string;
}

function printLabel(labelledObj: LabelledValue) {
  console.log(labelledObj.label);
}

let myObj = {size: 10, label: "Size 10 Object"};
printLabel(myObj);
```

### 可选属性
接口里的属性不全都是必需的。 有些是只在某些条件下存在，或者根本不存在。
```ts
interface SquareConfig {
  color?: string;
  width?: number;
}
```

### 接口索引
可以像数组一样使用
```ts
interface StringArray {
  [index: number]: string;
}

let myArray: StringArray;
myArray = ["Bob", "Fred"];

let myStr: string = myArray[0];
```

### 接口继承类
当接口继承了一个类类型时，它会继承类的成员但不包括其实现。
```ts
class Control {
    private state: any;
}

interface SelectableControl extends Control {
    select(): void;
}
```

## 泛型
```ts
// 泛型函数
function identity<T>(arg: T): T {
    return arg;
}

// 泛型接口
interface Pair<T, U> {
    first: T;
    second: U;
}

// 泛型类
class Box<T> {
    private value: T;

    constructor(value: T) {
        this.value = value;
    }

    getValue(): T {
        return this.value;
    }
}
```

### 泛型约束
```ts
interface Lengthwise {
    length: number;
}

function logLength<T extends Lengthwise>(arg: T): void {
    console.log(arg.length);
}

// 正确的使用
logLength("hello"); // 输出: 5

// 错误的使用，因为数字没有 length 属性
logLength(42); // 错误
```

## 异常处理
一样是`try-catch-finally`

## 命名空间
支持 `namespace` 语法的命名空间

## 异步
`async`函数必须返回`Promise`类型，类似C#的Task和Dart的Future。因为`Promise`和`async/await`是先后发布的，并且支持直接调用异步函数，所以存在这种包装类型的设计。而Kotlin/Swift的异步函数直接返回实际类型，不需要包装类型，但必须在异步上下文中调用（Kotlin的协程作用域，Swift的Task闭包）。