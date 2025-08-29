# 指针和数组
## 指针
指针是一个值为内存地址的变量。

指针有两个常用的符号：
1. &为取地址运算符
2. *为地址运算符，用于得到指针指向的变量，同时它也可以用在声明变量时表示是某种类型的指针类型

```c
int* p = &a; // p 是一个指针，指向变量 a 的内存地址
int b = *p; // b 是一个变量，它的值是 p 指向的变量 a 的值

// 相当于 b = a;
```

再看一个指针的经典例子：
```c
// 函数用于交换两个整数变量的值
void swap(int *a, int *b) {
    int temp = *a;  // 临时存储a指向的值
    *a = *b;        // 将b指向的值赋给a指向的位置
    *b = temp;      // 将临时存储的值赋给b指向的位置
}

int main() {
    int x = 10;
    int y = 20;

    printf("Before swap: x = %d, y = %d\n", x, y);
    // 函数传递参数时，传递的是地址，也就是指针，而不是变量本身
    swap(&x, &y);
    printf("After swap: x = %d, y = %d\n", x, y);

    return 0;
}
```

## 数组
数组名相当于数组首个元素的地址
```c
int arr[5];
int *ptr = arr;

// arr 也就是 &arr[0]

ptr + 2 == &arr[2] // 左右等价
*(ptr + 2) == arr[2] // 左右等价

// 两种赋值方式
arr[3] = 10;     // 数组下标访问
*(ptr + 3) = 10; // 指针算术访问
```

## 函数指针
```c
int add(int a, int b) {
    return a + b;
}

int main() {
    // 方式1：直接使用函数名
    int (*foo1)(int, int) = demo;   // 合法且推荐
    
    // 方式2：显式取地址
    int (*foo2)(int, int) = &demo;  // 同样合法但冗余
    
    // 两种方式相同
    foo1();
    foo2();
    
    return 0;
}
```

```c
// 使用 typedef ，可以把函数指针当作类型使用
typedef void(*Callback)();

void doSth(Callback callback) {
    // ...
    callback();
}
```

# 结构和联合
```c
struct User {
    int age;
    double height;
}

struct* ptr = &user;

// 两种方式都可以
ptr->age 
(*ptr).age
```

```c
// 联合，只能存储一个成员，所有成员共享同一块内存，因此大小等于最大成员的大小
union User {
    int age;
    double height;
}
```
