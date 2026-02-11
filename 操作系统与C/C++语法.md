## 变量和基本类型
使用new，可以把数据开辟到堆区，使用delete释放，new也可以用于基本类型，而不只是类。C++的类的构造函数一般不用new调用，会创建在栈区，使用new则创建在堆区。和Java不同，Java只要是类创建实例都会在堆区。

### 引用
```
int a = 1;
int& ref = a;
ref = 2; // a会等于2

int x = 1;
int y = 2;
int& ref = x;
ref = y; // 会让x等于2
```
对引用的所有操作都是在操作它引用的原本变量本身，引用的本质是一个指针常量，也就是不能修改指向地址的指针。

比较经典的引用使用例子：
```
void increment(int value) {
    value++;
}

int main(){
    int a = 1;
    // 并不能修改a
    increment(a)
}

// 可以使用指针
void increment(int* value) {
    (*value)++;
}
increment(&a);

// 或者使用引用
void increment(int& value) {
    value++;
}
increment(a);
```

### const
1. const和引用
```
// 如果a是常量，那么对它的引用也必须是常量
const int a = 1;
const int &f = a;

// 如果a是变量，那么对它的引用可以是也可以不是常量
int a = 1;
int &f = a;

// 给引用加上const，只会限制引用修改值，但a本身还是可以修改
int a = 1;
const int &f = a;

// 因为相当于
int a = 1;
const temp = a;
const int &f = temp;
```

2. const和指针
```
int x = 90;
int* a = new int;
*a = 2;
a = (int *) &x;

// 如果在 int* 前面加const，不能指针指向的内容的值，可以修改指针本身
int x = 90;
const int* a = new int; // 和 int const* a 一样，关键是const在 * 前还是后
*a = 2; // 错误
a = (int *) &x; // 没问题


// 如果在 int* 后面加const，让指针本身成为常量，不能指向新的地址，但可以修改指向的内容的值
int x = 90;
int* const a = new int;
*a = 2; // 没问题
a = (int *) &x; // 错误

// 前后都加const，则两者都不能修改
const int *const a = new int;
```

3. const和类
```
class Entity {
private:
    int x, y;
public:
    int GetX() const {
        x = 2; // 错误，const修饰的成员函数，不能修改成员属性，即const修饰的成员函数保证不会修改数据
        return x;
    }
};

void useEntity(const Entity &e) {
    // 如果GetX()没有const修饰，这里将会报错，因为参数为const引用，相当于承诺不会修改e，所以必须保证GetX()不能修改
    e.GetX();
}

// 但有一个mutable关键字可以避开这个限制
class Entity {
private:
    int x, y;
    mutable int z;
public:
    int GetX() const {
        z = 3; // 可以修改z，比如在一些调试的时候可以方便修改，但最好还是符合const的含义
        return x;
    }
};
```

### 常用关键字
1. constexpr
```
int a= 1;
const int x = a; // 没问题

int a= 1;
constexpr int x = a; // 错误，使用constexpr就强制使用常量来初始化
```

2. mutable
mutable还可以在lambda中使用，用处较少
```
int x = 8;
// 加上mutable才可以修改捕获的外部值（如果[]中不是=而是&则可以直接修改x）
auto f = [=]() mutable {
    x++;
};
// 但不会修改外部的x的值，所以x还是8，因为lambda的[]中使用的=号，是值传递
f();
```

3. typedef
```
// wages是double的同义词
typedef double wages;
```

4. using
```
// 可以用E作为Entity使用
using E = Entity;
```

5. decltype
得到参数（参数可以是变量也可以是函数，函数的话就用的函数的返回类型）的类型，可以用这个类型来声明变量
```
int a = 1;
decltype(a) c = 2; // c也是int类型
```

6. auto
C++的auto支持自动推导类型，比如有些类型很长，可以使用auto简化代码。

## 头文件
例如调用定义在其他翻译单元的函数，当前翻译单元必须声明该函数，如果很多个翻译单元都要使用该函数，就要在很多地方都要声明该函数，所以需要头文件。

* #pragma once：可以阻止单个头文件在一个翻译单元（比如一个CPP文件）中被多次包含，虽然自己不太会去包含一个头文件多次，但如果我们包含的头文件内部也包含了头文件，这就很难避免重复包含一个头文件，导致函数、变量重复定义的错误
* #define #ifdef #ifndef #endif：也有很多是使用这种方式
```
// Log.h
#ifndef _LOG_H
#define _LOG_H
...... 其他代码，只会被包含一次
#endif
```

> C的标准库头文件有.h的文件后缀，C++的标准库头文件没有后缀，C++的标准库作者这样设计来区分而已。

## C++支持运算符重载

## 类型转换
c++除了能使用c语言的强制类型转换外，还有自己增加的几种类型转换：
1. static_cast：相当于普通的强制类型转换，但不能保证安全。比如void*转为其他类型指针，又比如子类可以转为父类类型，但父类类型转为子类类型是不安全的。用于执行编译时类型转换，它在编译时进行类型检查，但不执行运行时检查。
```cpp
double d = 3.14;
int i = static_cast<int>(d); // 基本类型转换：double -> int

class Base {};
class Derived : public Base {};
Derived d;
Base* b = static_cast<Base*>(&d); // 上行转换：Derived* -> Base*
// Base* b = &d; 隐式转换为父类型也是安全的，所以一般会使用隐式转换
```
> **使用`static_cast`转换子类型为父类型，可以显式和隐式转换，而Java中只支持隐式转换**

2. dynamic_cast：专门用于处理多态继承中的向下转换和交叉转换（cross cast）。它在运行时检查转换的有效性和安全性。这是四个转换中唯一一个在运行时进行类型检查的。在类型层级关系之间转换，可以使用指针类型或者引用类型。指针类型转换失败返回空指针，引用类型转换失败会抛出std::bad_cast异常。**dynamic_cast 要求基类至少有一个虚函数（即必须是多态类型），因为运行时类型信息（RTTI）依赖于虚函数表。**
```cpp
class Base { virtual void foo() {} }; // 必须有虚函数（多态）
class Derived : public Base {};

Base* b = new Derived; // 可能指向Base，也可能指向Derived

// 运行时检查：如果b确实指向一个Derived对象，则转换成功
Derived* d = dynamic_cast<Derived*>(b);

if (d != nullptr) {
    // 转换成功，安全使用d
} else {
    // 转换失败，b并不指向Derived或其子类
}
```

3. const_cast：用于添加或移除变量的 const 或 volatile 限定符。它是唯一能修改 const 属性的类型转换操作符。**修改原本声明为 const 的对象是未定义行为，除非对象本身是通过非 const 指针/引用初始化的。**
```cpp
void print(char* str); // 一个旧的、不修改str的函数，但参数没声明为const

const char* greeting = "Hello";
// print(greeting); // 错误：不能将const char* 传递给char*
print(const_cast<char*>(greeting)); // 正确：去除了const

// 添加 const 属性（较少使用，因为通常可以隐式添加）
int x = 10;
const int* cx = const_cast<const int*>(&x);
```

4. reinterpret_cast：reinterpret 的意思是重新解释，属于粗暴的转换。用于低级别的、基于比特位的类型转换，直接将一种类型的位模式重新解释为另一种类型。它是最不安全的类型转换操作符。这是一种“我完全知道我在做什么”的转换。它不进行任何偏移量调整或类型检查，只是简单地将比特位模式重新解释。它的可移植性非常差，极度危险，应仅在非常底层的编程（如驱动、操作系统内核、自定义序列化）中不得已而为之使用。
```cpp
int* ip = new int(65);
char* cp = reinterpret_cast<char*>(ip); // 将int指针重新解释为char指针
```

## 类
c++保留了struct结构，主要就是为了兼容C语言，它和C++的类本质上一样，只是复合型数据。和基本类型一样，默认都是分配在栈中，使用new关键字则是分配在堆中。

### 构造与析构
```
class Entity {
public:
    float x, y;

    void print() {
        std::cout << x << "," << y << std::endl;
    }
};

int main() {
    Entity e; // 创建实例e，但没有明确调用构造函数初始化，实际就是调用默认构造函数，它内部的x和y是不确定的值
    e.print();
}
```
可以定义构造函数来初始化实例：
```
class Entity {
public:
    float x, y;

    Entity(int a, int b) {
        x = a;
        y = b;
    }    
};

int main() {
    Entity e = Entity(1, 2);
    e.print();
}
```
C++提供了删除构造函数的方式：
```
class Entity {
public:
    float x, y;

    Entity() = delete; // 比如删除默认构造函数，可以在一些场景使用，比如不希望有构造函数的情况
};
```

析构函数会在释放变量时被调用，它同时适用于栈和堆分配的对象（并非指堆中分配的对象，C/C++中的对象泛指分配了内存的变量，不区分栈和堆）。
```
class Entity {
public:
    float x, y;

    Entity() {};

    ~Entity() {}
};

void fun() {
    Entity e; // e在栈上分配，当函数fun的作用域结束，e就会被销毁，析构函数就会被调用。
}

int main() {
    fun();
}
```
比如构造函数中在堆上分配了对象，就可以在析构函数中释放。

### 继承
1. 虚函数
继承的函数，属于静态分派，也就是按声明类型调用方法；父类可以把函数设为虚函数，子类就可以重写父类的虚函数，变为动态分派，也就是按实际类型调用方法。

> C++ 11添加了override关键字，可以在子类重写的函数声明中添加，更明确地要求是重写了虚函数，如果父类没有同样的函数或者不是虚函数，都会报错。

2. 纯虚函数
不实现虚函数，并让它=0，即纯虚函数，有了纯虚函数，类就成为抽象类，此时Entity不能被实例化。子类实现了纯虚函数，才能实例化子类。
```
class Entity {
public:
    float x, y;

    Entity() {};
    virtual int aaa() = 0;
    ~Entity() {}
};
```

3. 虚析构函数
```
class Base {
public:
    Base() { std::cout << "Base constructor"; }

    ~Base() { std::cout << "Base destructor"; }
};

class Derived : public Base {
public:
    Derived() { std::cout << "Derived constructor"; }

    ~Derived() { std::cout << "Derived constructor"; }
};

int main() {
    Base *base = new Base();
    delete base;

    Derived *derived = new Derived();
    delete derived;

    Base *base2 = new Derived();
    delete base2;
}

// 打印：
// Base constructor
// Base destructor

// Base constructor
// Derived constructor
// Derived destructor
// Base destructor

// Base constructor
// Derived constructor
// Derived destructor
```
可以看到，前两种情况正常，但第三种没有调用子类的析构函数，这样可能造成内存泄漏。原因同样是C++默认是根据声明的类型静态调用，所以需要使用虚析构函数
```
class Base {
public:
    Base() { std::cout << "Base constructor"; }

    virtual ~Base() { std::cout << "Base destructor"; }
};
```

虚析构函数，子类的析构函数才能被执行


3. 接口
如果一个类只有纯虚函数，那么它就相当于接口，不过C++中没有专门的接口，只有纯虚函数的类就相当于接口。

### 成员初始化列表
初始化成员变量，也可以在构造函数中使用初始化列表：
```
class Example {
public:
    Example() {
        std::cout << "created" << std::endl;
    }

    Example(int x) {
        std::cout << "created with " << x << std::endl;
    }
};

class Entity {
public:
    Example example;

    // Entity() {
    //     example = Example(1);
    // };

    Entity() : example(Example(1)) {
        //...
    };
};

int main() {
    Example example;
}
```
好处是可以把构造函数中初始化变量和其他逻辑分开，更加清晰。而更重要的区别是，可以避免一次不必要的构造，如果像上面注释掉的方式，example会先赋值为使用默认构造函数构造的值，再赋值为使用有参构造函数构造的值，而使用成员初始化列表，就只会构造一次。所以尽量使用成员初始化列表

### 隐式构造函数与explicit
C++的构造函数支持隐式调用：
```
class Entity {
public:
    int x;

    Entity(int a) : x(a) {
    };
};

int main() {
    Entity a = 2; // 相当于 Entity e = Entity(2);
    Entity b(2); // 另一种调用构造函数的方式
}
```

explicit可以禁止隐式调用：
```
class Entity {
public:
    int x;

    explicit Entity(int a) : x(a) {
    };
};

int main() {
    Entity a = 2; // 报错
}
```
使用explicit可以避免意外的隐式调用。

### 栈和堆中创建对象
栈中的变量在代码块作用域结束后就会自动销毁，如果我们想要在作用域结束后继续使用该变量，就要在堆上分配内存，C++提供了new关键字。（Java只能在堆上创建类的实例）。
```
int main() {
    Entity entity = Entity(); // 在栈中分配，作用域结束就会自动销毁

    Entity *e;
    { // C++的代码块就是一个作用域
        Entity *entity = new Entity(); // 在堆中分配
        e = entity;
    }
    // 此时e没有销毁，执行下一行代码才销毁
    delete e; // 执行delete后销毁
}
```

C++中写了new，就表示要在堆上分配内存，不管它是一个类、还是基本类型、或者数组：
```
int* a = new int; // 4个字节
int* b = new int[50]; // 200字节
Entity* e = new Entity[50]; // 50个Entity实例的字节大小
```

栈作用域生命周期：
```
int main() {
    {
        Entity e; // 执行Entity的构造函数
    } // 执行Entity的析构函数
}
```

堆作用域生命周期：
```
int main() {
    {
        Entity e = new Entity; // 执行Entity的构造函数
    }
    // 不会销毁e，直到调用 delete e;
}
```

一个自定义的作用域指针：作用域结束自动销毁堆上的对象：
```
class ScopedPtr {
private:
    Entity *ptr;
public:
    ScopedPtr(Entity *ptr) : ptr(ptr) {}

    virtual ~ScopedPtr() {
        delete ptr;
    }
};

int main() {
    {
        ScopedPtr e = new Entity(); // 隐式构造
    } // e结束作用域，就会销毁内部的Entity
}
```

### 智能指针
智能指针可以帮助我们自动调用new和delete来管理堆上的对象。
1. unique_ptr
作用域指针，作用域结束就会调用delete销毁。

```
int main() {
    {   
        // unique_ptr的构造函数使用了explicit，所以不能使用 std::unique_ptr<Entity> entity = new Entity();
        std::unique_ptr<Entity> entity(new Entity());
        // 或者使用make_unique来构造
        std::unique_ptr<Entity> entity = std::make_unique<Entity>();
        entity->print(); // unique_ptr重载了->运算符，所以可以直接使用它来调用Entity的函数

        // 报错，不能赋值
        std::unique_ptr<Entity> e0 = entity;
    }
}
```

unique_ptr的拷贝构造函数和拷贝操作符都被删除了，就是为了避免错误的使用
```
unique_ptr(unique_ptr const&) = delete;
unique_ptr& operator=(unique_ptr const&) = delete;
```
unique_ptr不能被复制，因为如果两个unique_ptr指向同一个内存块，当其中一个unique_ptr结束作用域就会释放该内存，那么另一个unique_ptr就指向了已经被释放的内存。所以unique_ptr限制

2. shared_ptr
引用计数，
```
int main() {
    std::shared_ptr<Entity> e0;
    {
        std::shared_ptr<Entity> e = std::make_shared<Entity>(); // 引用计数为1
        // 或者这样构造
        // std::shared_ptr<Entity> e(new Entity());

        e0 = e; // 引用计数为2
    } // e的作用域结束了，但是这个Entity并不会销毁，因为e0仍然引用它。引用计数为1
    // e0的作用域是main函数，所以main函数结束后，才会销毁这个Entity，并执行析构函数
}
```
> std::weak_ptr不会增加引用计数

### 对象的拷贝构造、拷贝赋值
```
int a = 2;
int b = a; // a的值复制到了b，a和b是两个独立的变量，占用不同的内存地址
```
结构或者类也同样如此：
```
Entity e = Entity();
Entity e1 = e; // e和e1是两个独立的变量，占用不同的内存地址
```

如果使用new的话：
```
Entity* e = new Entity();
Entity* e1 = e; // 复制的是指针，所以e和e1指向同样的内存地址
```
**当使用=等号赋值操作符的时候，总是在复制值（引用除外），使用指针的时候复制的是指针，因为值就是指针**

这里看一个自定义的String类，理解C++中的复制操作：
```
class String {
private:
    char *buffer;
    unsigned int size;
public:
    String(const char *string) {
        size = strlen(string);
        buffer = new char[size + 1]; // size + 1，因为字符串最后需要一个0表示结束
        memcpy(buffer, string, size);
        buffer[size] = 0;
    }

    ~String() {
        delete[] buffer; // 销毁时堆上的buffer
    }

    // 由于<<左边的对象是一个ostream对象，<<右边的是类对象，而能当作类的成员函数的左边必须是类对象，
    // 所以重载<<要么在类外部声明，要么要使用friend友元。（不过这个细节不是这个代码例子的关键）
    friend std::ostream &operator<<(std::ostream &stream, const String &string);
};

std::ostream &operator<<(std::ostream &stream, const String &string) {
    stream << string.buffer;
    return stream;
}

int main() {
    {
        String s1 = "first";
        String s2 = s1; // 因为C++中的复制赋值，会把s1中的buffer和size都复制给s2，导致s1和s2中的buffer指针指向同一个内存地址（浅拷贝）
    } // 发生异常：作用域结束，两个变量都会销毁，调用析构函数，同一个buffer被delete两次
}
```

1. 拷贝构造函数
所以这种情况需要深拷贝，我们可以写一些函数来深拷贝，但C++提供了拷贝构造函数，它是构造函数的一种，在把一个类变量复制给另一个类变量的时候会被调用
```
class String {
    // ...... 省略其他代码

    // C++默认的拷贝构造函数，它会浅拷贝类中所有的成员属性
    String(const String &other) : buffer(other.buffer), size(other.size) {

    }

    // 拷贝构造函数可以删除，比如unique_ptr就删除了拷贝构造函数，禁止赋值
    String(const String &other) = delete;

    // 自定义拷贝构造函数，size不是指针，可以直接复制，而buffer则需要深拷贝
    String(const String &other) : size(other.size) {
        buffer = new char[size + 1];
        memcpy(buffer, other.buffer, size + 1);
    }
};
```
重写了拷贝构造函数后，就可以正常将一个String变量赋值给另一个String变量。

但如果像这样使用：
```
// 因为C++的函数参数默认都是值传递，所以这个函数传值过程，也会有复制过程，产生不必要的开销
void printString(String string) {
    std::cout << string << std::endl;
}

// 应该使用引用，而且如果不需要修改值的话，可以加上const
void printString(const String& string) {
    std::cout << string << std::endl;
}
```

2. 拷贝赋值运算符
和拷贝构造函数类似
```
class String {
    // ...... 省略其他代码
    
    // 自定义拷贝赋值运算符
    String &operator=(const String &other) {
        if (this != &other) {
            delete[] buffer;
            size = other.size;
            buffer = new char[size + 1];
            memcpy(buffer, other.buffer, size + 1);
        }
        return *this;
    }
};

int main() {
    String s1 = "11111"; // 构造函数
    String s2 = s1; // 拷贝构造函数
    String s3 = "33333";
    s3 = s1; // 拷贝赋值运算符
}
```

* 拷贝构造函数使用传入对象的值生成一个新的对象的实例，而拷贝赋值运算符将对象的每个成员变量的值赋值给一个已经存在的实例。
* 将对象传递给函数的形参、或者函数返回一个对象，都会执行拷贝构造函数，因为这些情况都会创建临时对象实例。如果返回的对象是引用的方式，那么会执行拷贝赋值。

## 左值和右值
```
int getValue() {
    return 10;
}

int main() {
    // i是一个有内存地址的实际变量；而5是一个数字字面量，没有内存地址存储空间
    // 左值就是有实际内存地址的变量，右值没有内存地址的临时值
    // 可以把右值赋值给左值，但不能给右值赋值，比如不能让 5 = i;
    int i = 5;
    int a = i; // 这里i的值5就是临时的右值

    // 类似的，函数返回的临时值也是右值，可以把它赋值给左值，但是不能 getValue() = i;
    int b = getValue();

}
```

但函数可以返回一个左值引用，此时就可以对它赋值了，因为返回的左值引用就是一个有内存地址的变量
```
int& getValue() {
    static int value = 10;
    return value;
}

int main() {
    getValue() = 5;
}
```

当函数的参数类型是左值引用时，也必须传递一个左值：
```
void setValue(int& value) { }
// void setValue(int &value) { } 这样写也一样

int main() {
    int i = 10;
    setValue(i);
    setValue(10); // 错误
}
```

但可以使用const来达到传递右值也可以使用的目的：
```
void setValue(const int& value) {

}

int main() {
    // 因为使用const之后，const int& a = 10;
    // 相当于：
    // int temp = 10;
    // const int& a = temp;
    setValue(10); // 没问题
}
```

类似的，在平时的开发中，就可以这样使用：
```
void printName(std::string& name) {}

// 很多函数写const引用，就是为了方便调用时直接传递右值
void printName2(const std::string& name) {}

int main() {
    std::string firstName = "aaa";
    std::string lastName = "bbb";
    std::string fullName = firstName + lastName;
    printName(fullName); // 正确
    printName(firstName + lastName); // 错误，右值不能作为左值引用
    printName2(firstName + lastName); // 正确，右值可以作为const左值引用
}
```

但C++还可以写一个接受右值引用参数的函数，它只接受临时值（右值）：
```
// 两个&符号，表示右值引用
void printName(std::string&& name) {

}

int main() {
    std::string firstName = "aaa";
    std::string lastName = "bbb";
    std::string fullName = firstName + lastName;
    printName(fullName); // 错误，不接受左值
    printName(firstName + lastName); // 正确，使用右值
}
```
**只接受右值引用，也就是只接受临时值，那么我们就不用担心这个右值引用的生命周期，也不用担心它的数据信息是否需要保持完整，因为我们知道右值引用是临时变量**

右值引用在C++的移动操作中用处较大。

### 对象的移动
在C++中，当我们把一个对象传递给一个函数，此时会拷贝该对象，从函数返回一个对象时也是如此，需要在函数中创建对象，然后复制给返回值（现在C++编译器在一些情况会做返回值优化，可以避免这种情况）。可以感受到，C++中存在很多拷贝的操作。

看一个例子：
```
class String {
private:
    char* buffer;
    unsigned int size;
public:
    String() = default;

    String(const char* string) {
        size = strlen(string);
        buffer = new char[size + 1];
        memcpy(buffer, string, size);
        buffer[size] = 0;
    }

    String(const String& other) : size(other.size) {
        buffer = new char[size + 1];
        memcpy(buffer, other.buffer, size + 1);
    }

    ~String() {
        delete[] buffer;
    }
};

class Entity {
private:
    String name;
public:
    Entity(const String& name) : name(name) {}
};

int main() {
    // 相当于 Entity entity(String("aaaa"));
    Entity entity("aaaa");
}
```
这里会先调用一次String的构造函数，发生一次字符数组的复制，然后调用String的拷贝构造函数初始化Entity内部的name字段，又发生了一次字符数组的复制。

#### 移动构造函数
此时，我们可以使用**移动构造函数**
```
class String {
private:
    char *buffer;
    unsigned int size;
public:
    String() = default;

    String(const char *string) {
        size = strlen(string);
        buffer = new char[size + 1];
        memcpy(buffer, string, size);
        buffer[size] = 0;
    }

    String(const String &other) : size(other.size) {
        buffer = new char[size + 1];
        memcpy(buffer, other.buffer, size + 1);
    }

    // 移动构造函数会“窃取”资源，直接浅拷贝
    String(String&& other) : size(other.size) {
        buffer = other.buffer;
        other.size = 0;
        other.buffer = nullptr;
    }

    ~String() {
        delete[] buffer;
    }
};

class Entity {
private:
    String name;
public:
    Entity(const String &name) : name(name) {}

    // 或者：Entity(String &&name) : name((String &&) name) {}，但一般还是用std::move
    Entity(String &&name) : name(std::move(name) ) {}
};

int main() {
    Entity entity("aaaa");
}
```
此时，还是会调用一次普通的构造函数，产生一次复制，但是会调用Entity接受右值引用的构造函数，这里就会调用String的移动构造函数，只是浅拷贝，所以总共只发生了一次字符数组的复制。

#### 移动赋值运算符
前面了解了接受左值引用的拷贝赋值运算符，那么也有只接受右值引用的移动赋值运算符。

移动构造函数针对要创建新对象的情况，而移动赋值运算符针对的是赋值现有的变量：
```
class String {
    // ......省略其他代码

    String &operator=(String &&other) noexcept {
        if (this != &other) {
            delete[] buffer;
            size = other.size;
            buffer = other.buffer;

            other.size = 0;
            other.buffer = nullptr;
        }
        return *this;
    }
};

int main() {
    String a = "aaaaa"; // 调用普通构造函数
    String b = "bbbbb"; // 调用普通构造函数
    b = std::move(a); // 调用移动赋值运算符
    b = String("xxx"); // 调用移动赋值运算符
}
```
可以看出来，对于右值的情况，可以自定义移动赋值运算符，从而支持直接通过右值来赋值，减少复制次数。

> 拷贝赋值运算符需要使用左值赋值，而移动赋值运算符针对的是右值。

> 如果一个类，使用这个类的时候不是通过指针传递，就会发生复制或者移动，如果它内部的属性有指针，就会产生浅拷贝的问题，此时就需要自定义拷贝、移动之类的操作。但如果使用类都是通过指针传递，多个指针都是指向同一个对象，就没有所谓的复制，比如Java中的类的实例（引用类型）就相当于都是使用指针，所以没有复制的问题。

## 命名空间
命名空间可以避免同名的函数、变量冲突。比如C语言中没有命名空间，要避免冲突就要给函数名称加上前缀。
namespace apple {
    void test() {  }

    // 命名空间可以嵌套
    namespace functions {
        void print() {  }
    }
}

int main() {
    // 不使用命名空间，就要用全称
    apple::test();

    // 使用命名空间
    using namespace apple;
    test();

    // 只使用命名空间内的某个指定的函数
    using apple::test;
    test()

    apple::functions::print();

    using namespace apple::functions;
    print();
}

## 宏
C++宏在预处理阶段使用，发生在编译之前。宏可以将代码中的文本替换成其他，并且还可以使用参数。

一些简单的例子（实际使用不一定适合，只是体现一下宏的作用）：
```
替换为代码
#define WAIT std::cin.get()

int main() {
    WAIT;
}
```
替换为{
```
#define OPEN_CURLY {

int main() OPEN_CURLY

}
```
替换为代码，可以传参
```
#define LOG(x) std::cout<<(x)<<std::endl

int main() {
    LOG("hello");
}
```
DEBUG时，替换为代码，否则替换为空
```
#define DEBUG 0
#if DUBUG == 1
#define LOG(x) std::cout<<(x)<<std::endl
#else
#define LOG(x)
#endif
```

## 模板（类似泛型）
C++的模板比C#/Java中的泛型功能更强大，C#/Java中的泛型受限于类型系统，而C++的模板更像是宏，

模板分为`函数模板`和`类模板`
1. 函数模板
```
template<typename T>
//或者 template<class T>
void swap(T &a, T &b){
    T temp = a;
    a = b;
    b = temp;
}
```
编译器在发现代码调用模板函数的时候，才会生成各种实际类型的函数代码

2. 类模板
模板还可以使用数据类型的值，比如这里会产生一个长度为5的Array类：
```
template<typename T, int N>
class Array {
private:
    T array[N];
public:
    int getSize() const { return N; }
};

int main() {
    Array<int, 5> array;
}
```
类模板没有自动类型推导，创建对象时，类型需要写上尖括号和类型。且类模板的模板类型可以写默认类型，类模板的成员函数在运行时才创建