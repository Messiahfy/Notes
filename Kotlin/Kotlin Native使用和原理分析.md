# 前言

2017 年的 Google I/O 大会，Google 宣布 Kotlin 成为 Android 开发的官方支持语言，接着在 2019 年的 Google I/O 大会，Google 提出了 “Kotlin First” 的理念，Kotlin 取代 Java 成为 Android 平台的主流开发语言。由于 Kotlin 拥有更现代化的语法设计和使用体验，并且几乎无缝兼容 Java，语言转换成本很低，作为 Java 虚拟机体系的一种编程语言，Kotlin 也因此被很多人认为是 “Better Java”。


但其实在 2017 年的 KotlinConf 大会上，JetBrains 就已经开始宣传 Kotlin 多平台的预览版（当时还没有发布正式版），经过数年发展，Kotlin 多平台在2023年发布稳定版本，Kotlin 的使用场景逐渐从 Java 虚拟机扩展到其他平台，当前 Kotlin 官网关于多平台的介绍图如下：

![Image](./引用图片/Kotlin%20Native/Kotlin多平台介绍.png)

Kotlin 支持运行在所有主流平台，从偏技术的角度，Kotlin 支持的目标平台如下：

1. JVM（Java虚拟机）：编译为 Java 字节码，运行在**服务器**和 **Android 客户端**的 Java 虚拟机上
2. Web：编译为 JavaScript、Wasm，运行在浏览器上
3. Native：编译为 Native 库和可执行文件，原生运行在 Linux（包括Android）、Windows 和 Apple 各系统（iOS、macOS等）

![Image](./引用图片/Kotlin%20Native/Kotlin多平台结构图.png)

在 Kotlin\-JVM 场景，Kotlin 将编译为 Java 字节码运行在 Java 虚拟机中，只要了解 Java 语言和 Java 虚拟机就很容易理解它的使用方式和运行机制。而 Kotlin\-JS 则更为直接，因为 JavaScript 是脚本语言，它的源码会直接交给解释引擎来执行，所以 Kotlin\-JS 会把 Kotlin 源码转换为 JavaScript 源码，这对于开发者很容易理解，并且 JavaScript 和Kotlin 的语法差别不大，跨语言调用也比较简单。


本文的主要介绍对象为 Kotlin\-Native，它是 Kotlin Multiplatform（KMP）技术栈的组成部分之一，它的目标是将 Kotlin 代码编译为原生二进制文件，使其可以脱离 Java 虚拟机，直接在 iOS、macOS、Linux、Windows 等平台上运行。


Kotlin Native 的上手会比 Kotlin\-JVM 和 Kotlin\-JS 稍微麻烦一点，因为使用 Kotlin\-Native 涉及 Native 开发环境，需要对 C/C\+\+ 以及 Linux（或者Mac/Win）开发环境具备基本的了解，并且 Kotlin 和 C 的语法差异很大，Kotlin 和 C 之类的 Native 语言和编译产物交互也就会复杂一点。


## 内容介绍

下面将分析这几个方面：

1. Kotlin Native 的使用方式

2. 简要了解 Kotlin Native 的编译方式

3. C 语言与 Kotlin 编译生成的 Native 二进制文件的交互和原理（**需要了解 Linux 平台的 ELF 文件格式**）

4. klib 文件


建议：
1. 使用 Kotlin Native 必然会涉及到 Native 的开发知识和概念，比如编译链接、ELF文件（Linux场景为例）等，所以拥有 Native 开发的基本知识仍然是必要的。

2. 具备基本的 Native 开发知识后，Kotlin Native 的使用上手非常简单，所以更重要的是了解 Kotlin Native 实现跨语言交互的原理，即使没有选择使用 Kotlin Native 技术或者过几天就忘了用法，但也可以通过对它的学习，增加一些对其他跨平台技术的思考和理解。


# 使用方式

在实际的开发工作中，我们一般会使用集成开发环境和插件来开发 Kotlin Native 项目，而集成开发环境和插件内部会调用相关命令工具帮我们完成编译构建工作，所以本文档选择直接使用命令行工具完成所有操作，以便更直观地理解 Kotlin Native 的各种使用方式，这样更利于我们在使用 IDE 和插件开发时，掌握各种配置背后的实际工作过程。 


## 下载编译工具

Kotlin Native 提供的编译工具可以在官方 github 页面下载：https://github\.com/JetBrains/kotlin/releases

![Image](./引用图片/Kotlin%20Native/Kotlin%20Native开发工具下载.PNG)

它提供了 Linux、macOS（ARM、x86两种版本）和 Windows 三种桌面平台的开发工具，比如我们大都使用 M 系列芯片的 MacBook，那么可以选择下载 `macos-aarch64` 这个版本。

> 如果你已经使用 Android Studio 或者 IDEA 安装过 Kotlin Multiplatform 插件，可以不用再手动下载Kotlin Native的编译工具，因为该插件会包含 Kotlin Native 编译工具，它在 macOS 上的目录为：\~/.konan/kotlin-native-prebuilt-macos-aarch64-2.1.10/  （我本地当前的版本是 2.1.10）

编译工具的 `bin` 目录包含了可以使用的命令行工具：

![Image](./引用图片/Kotlin%20Native/Kotlin%20Native开发工具bin目录.PNG)

其中有3个工具是在 Kotlin Native 场景常用的：

1. kotlinc\-native：把 kotlin 编译为 native 二进制文件的核心工具，相当于 C 语言的 gcc/clang 命令工具

2. cinterop：根据 C 语言声明函数/全局变量生成包含桥接函数和变量的`klib`，用于 Kotlin 调用 C 语言

3. klib：可以便捷的解析`klib`文件的内容



## 生成可执行文件

现在我们就开始使用 Kotlin Native 编译工具，把 Kotlin 源文件编译为 native 可执行文件，看看实际操作。


1. 首先，编写了一个 `foo.kt` 源码文件，内容就是一个函数：

```Kotlin
fun foo(a: Int, b: Int):Int {
    return a + b
}
```



2. 再编写了一个 `hello.kt` 源码文件，`main` 函数内调用了 `foo` 函数：

```Kotlin
fun main() {
    println("Hello, Kotlin/Native!")
    println("call foo result：${foo(1, 2)}") // 调用 foo 函数
}
```



3. 使用 kotlinc\-native 命令工具，编译 `hello.kt `和 `foo.kt`：

```C
kotlinc-native hello.kt foo.kt 
```

因为没有使用额外的参数指定编译产物类型和名称，**默认就会生成名为****`program`****的可执行文件，在 macOS 和 Linux 上，生成的可执行文件扩展名为 \.kexe（扩展名可以删掉，没有影响），windows上则为 \.exe**



> 和 clang/gcc 编译多个源码的方式类似，需要把所有参与编译的源文件都列在命令行中，比如上面👆的例子，就把 hello\.kt 和 foo\.kt 都写上了（也可以使用通配符方式包含整个目录）


4. 执行生成的可执行文件：

```Plain Text
./program.kexe


// 打印结果：
Hello, Kotlin/Native!
call foo result：3
```

经过以上流程，我们已经使用 Kotlin Native 工具把 Kotlin 源文件编译为了 Native 可执行文件，并成功运行。如果是在 macOS 上操作，这个可执行文件的格式就是 Apple 平台的 Mach\-O 格式（在 macOS 上，C/C\+\+/OC/Swift编译生成的可执行文件也是 Mach\-O 格式），如果是 Linux 则生成 ELF 文件。

### 指定产物名称

如果想指定产出文件名称，可以使用 `-o` 选项：

```Kotlin
kotlinc-native hello.kt ./foo/foo.kt  -o myProgram
```

在 macOS/Linux 上会生成 myProgram\.kexe 文件


### 入口函数

kotlinc\-native 编译可执行程序时，默认入口函数为`root package`的`main`方法，没有指定`package`的 `kotlin`源文件都属于 `root package`，否则属于指定的`package`。



比如在`entryTest.kt`文件中设置了`package`：

```Kotlin
package entry.test

fun main() {
    println("set package")
}
```



此时编译会报错，因为`root package`并没有`main`函数，而是在`entry.test package`中：

```Plain Text
kotlinc-native entryTest.kt

// 报错：error: could not find 'main' in '<root>' package.
```



此时需要使用 \-e 选项，指定入口函数为`entry.main`：

```Kotlin
kotlinc-native entry.kt -e entry.test.main
```



入口函数也可以不叫`main`：

```Kotlin
package entry.test

fun customEntry() {
    println("custom entry")
}
```

同样使用 \-e 选项，指定入口函数为`customEntry`：

```Kotlin
kotlinc-native entry.kt -e entry.test.customEntry
```



kotlinc\-native 还有其他一些命令选项，可以使用 kotlinc\-native \-h 查看。

使用 kotlinc\-native \-X 可以查看扩展选项，扩展选项是一些不太常用的更接近底层的一些配置。比如使用 `-Xprint-bitcode`可以打印出编译的 Kotlin 源文件对应的 LLVM bitcode，`-Xdump-perf=<path>` 会打印编译过程的性能报告到指定路径的文件中。



## 指定目标平台

前面的例子中，使用 kotlinc\-native 时没有指定目标平台，默认就会生成当前平台的可执行文件，比如当前如果是 macOS 平台，就会生成 Mach\-O 格式的可执行文件。我们也可以指定目标平台，生成其他平台的可执行文件。



使用 `-list-targets` 选项查看 Kotlin Native 支持的平台：

```C
kotlinc-native  -list-targets

// 输出如下：
linux_x64
linux_arm32_hfp (deprecated)
linux_arm64
mingw_x64
android_x86
android_x64
android_arm32
android_arm64
macos_x64
macos_arm64 (default)
ios_arm64
ios_x64
ios_simulator_arm64
watchos_arm32
watchos_arm64
watchos_x64
watchos_simulator_arm64
watchos_device_arm64
tvos_arm64
tvos_x64
tvos_simulator_arm64
```

可以看到有 Linux\_x64、linux\_arm64、mingw\_x64（也就是windows）、Android、Mac、iOS等各种操作系统，支持了所有主流的桌面和移动端操作系统。



使用 `-target <platform_name>` 选项可以选择产物的目标平台，试试生成一个 Linux ARM64 的可执行文件：

```kotlin
// 源文件
fun main() {
    println("Hello, Linux ARM64 Kotlin/Native!")
}
```

```kotlin
// 编译
kotlinc-native hello.kt -target linux_arm64
```

将生成 Linux 平台的 ELF 格式的可执行文件。



## 指定产物类型

Kotlin Native 不仅可以把 Kotlin 源文件编译为各种目标平台的可执行文件，还可以编译为静态库、动态库、klib等产物，使用 `-p`选项可以设置产物的类型。



### 可执行文件

kotlinc\-native 默认情况就会生成可执行文件，我们也可以明确地使用`-p program`指定生成可执行文件：

```C
kotlinc-native hello.kt -p program -o hello_program
```

当前平台为 macOS，所以会生成 `hello_program.kexe`。如果指定目标平台，则会生成其他平台的可执行文件。



### 静态库

Linux 和 macOS 静态库都是扩展名为`.a` 的文件，它是`.o`可重定位文件的包装格式。



使用`-p static`可以把 Kotlin 源码编译生成静态库。

```Plain Text
kotlinc-native foo.kt -target linux_arm64 -p static -o static_foo_linux_arm64
```

比如这里指定目标平台为 Linux ARM64，命名为`static_foo_linux_arm64`，此时将生成名为`libstatic_hello_linux_arm64.a`的静态库文件，和相应的头文件`static_foo_linux_arm64_api.h`



### 动态链接库

使用`-p dynamic`会生成动态链接库：

```C
kotlinc-native foo.kt -target linux_arm64 -p dynamic -o dynamic_foo_linux_arm64
```

比如这里指定目标平台为 Linux ARM64，命名为`dynamic_foo_linux_arm64`，执行命令后将会生成`libdynamic_foo_linux_arm64.so`动态链接库文件，和头文件`dynamic_foo_linux_arm64_api.h`



类似的，如果目标平台为 macOS 则生成 `.dylib`和头文件，目标平台为 windows 则生成`.dll`和头文件



### klib

klib 是 Kotlin 多平台场景的二进制库的发布格式，与Java的Jar文件类似，它本质上也是一个Zip文件，在后文将介绍它的具体内容。



使用`-p library`可以生成 klib：

```C
kotlinc-native foo.kt -p library -o library_foo
```

将生成 `library_foo.klib`



### framework

Kotlin 也可以编译为 Apple 平台的 framework 库，使用`-p framework`：

```C
kotlinc-native hello.kt -p framework -o framework_foo_macos_arm64
```

会生成`framework_foo_macos_arm64.framework`文件，和相应的 dSYM 文件



> 本来还有一个 bitcode 选项，可以将 Kotlin 源码编译生成 LLVM bitcode，但现在已经不能使用 https://kotlinlang\.org/docs/native\-faq\.html\#how\-do\-i\-enable\-bitcode\-for\-my\-kotlin\-framework
> 
> 



## 使用产出物

使用方式在官方文档有介绍：https://kotlinlang\.org/docs/native\-dynamic\-libraries\.html



以生成Linux的动态链接库为例，再另外写一个 test\.c 文件，引入生成的动态库的头文件，调用头文件中提供的函数：

```Kotlin
#include "dynamic_foo_linux_arm64_api.h"
#include <stdio.h>

int main(int argc, char **argv)
{
    dynamic_foo_linux_arm64_ExportedSymbols *libfoo = dynamic_foo_linux_arm64_symbols();
    int result = libfoo->kotlin.root.foo.foo(1, 2);
    printf("c call kotlin dynamic lib result: %d\n", result);
    return 0;
}
```

然后在 Linux ARM64 平台完成编译，然后就可以成功运行

```Kotlin
clang test.c ./libdynamic_foo_linux_arm64.so
```



测试静态库的情况，需要链接C\+\+标准库才能执行，因为 Kotlin Native Runtime 库是用c\+\+写的

```Kotlin
clang test_static.c ./libstatic_foo_linux_arm64.a -lstdc++
```



# 编译流程简介

Kotlin Native 和当前很多语言一样（比如Swift），都采用了 LLVM 编译器框架，Kotlin 编译器前端将 Kotlin 源码编译为 Kotlin IR中间形式，然后转换为 LLVM IR，从而复用 LLVM 编译器后端功能将 LLVM IR 编译为最终的二进制文件。

![编译流程](./引用图片/Kotlin%20Native/Kotlin%20Native编译流程.png)

klib 是 Kotlin 的库形式，自然也可以参与编译，Kotlin Runtime 则是 Kotlin Native 的运行时库，用于提供 Kotlin Native 代码的运行时环境，



**关于 Runtime：**

C\+\+ 和 Objective\-C 等语言都一个 Runtime，为什么它们需要 Runtime？简单来说，比如它们的面向对象语法，编译为汇编或者机器指令后是什么呢，毕竟汇编和机器指令中根本没有类和对象的概念。实际上，这些复杂语法都需要额外编写代码来创造这些概念，比如可以先用 C 语言来定义类在内存中的结构形式、对象的内存布局、继承/多态等特性，把这些基础代码作为库，这个库就称为 Runtime，编译器在编译 C\+\+ 和 Objective\-C 等语言的时候，很多语法都会被编译为调用 Runtime API 的形式，比如创建一个对象，编译后的指令就是调用 Runtime 中已经写好的指令来完成。Kotlin 的 Runtime 也是类似的。



C 语言可以看作是机器指令或汇编的简单映射，它只有基本的流程控制和数据类型，这些都可以直接地转换为机器指令存在的形式，所以 C 语言其实不需要 Runtime。不过在 C 语言编译为可执行文件的时候，还是会链接一些库，比如 main 函数并不是 C 代码执行的实际入口，操作系统中会有一个库提供实际的入口，这个实际入口会执行获取参数表、环境变量等指令，然后将参数传给 main 函数，再从 main 函数开始执行 C 代码编译后的指令。但可以认为这些库只是简单的辅助作用，不算是Kotlin/C\+\+/OC这些语言中那种复杂的 Runtime。


# C调用Kotlin的方式和原理

在前面的【指定产物类型】段落中，介绍了把 Kotlin 源文件编译为静态库/动态库和相应的头文件，所以我们可以在 C 源文件中引入头文件，调用头文件声明的函数和变量，然后和 Kotlin 生成的静态库/动态库完成链接，就可以实现 C 调用 Kotlin 代码。


## 使用方式

下面来实际操作一下：

1. 编写 Kotlin 源文件：

```kotlin
fun foo(a: Int, b: Int): Int{
    return a + b
}

class MyTestClass {
    var testInt: Int = 0
    var testDouble: Double? = null 
}
```



2. 将 Kotlin 源码编译为静态库，因为打算在 Linux 平台测试（方便分析ELF文件），所以指定 linux\_arm64：

```Shell
kotlinc-native mytest.kt -target linux_arm64 -p static -o static_mytest_linux_arm64
```

将生成`libstatic_mytest_linux_arm64.a`静态库和相应的头文件`static_bar_linux_arm64_api.h`

```c  
#ifndef KONAN_STATIC_MYTEST_LINUX_AMR64_H
#define KONAN_STATIC_MYTEST_LINUX_AMR64_H
#ifdef __cplusplus
extern "C" {
#endif
#ifdef __cplusplus
typedef bool            static_mytest_linux_amr64_KBoolean;
#else
typedef _Bool           static_mytest_linux_amr64_KBoolean;
#endif
typedef unsigned short     static_mytest_linux_amr64_KChar;
typedef signed char        static_mytest_linux_amr64_KByte;
typedef short              static_mytest_linux_amr64_KShort;
typedef int                static_mytest_linux_amr64_KInt;
typedef long long          static_mytest_linux_amr64_KLong;
typedef unsigned char      static_mytest_linux_amr64_KUByte;
typedef unsigned short     static_mytest_linux_amr64_KUShort;
typedef unsigned int       static_mytest_linux_amr64_KUInt;
typedef unsigned long long static_mytest_linux_amr64_KULong;
typedef float              static_mytest_linux_amr64_KFloat;
typedef double             static_mytest_linux_amr64_KDouble;
typedef float __attribute__ ((__vector_size__ (16))) static_mytest_linux_amr64_KVector128;
typedef void*              static_mytest_linux_amr64_KNativePtr;
struct static_mytest_linux_amr64_KType;
typedef struct static_mytest_linux_amr64_KType static_mytest_linux_amr64_KType;

typedef struct {
  static_mytest_linux_amr64_KNativePtr pinned;
} static_mytest_linux_amr64_kref_kotlin_Byte;
typedef struct {
  static_mytest_linux_amr64_KNativePtr pinned;
} static_mytest_linux_amr64_kref_kotlin_Short;
typedef struct {
  static_mytest_linux_amr64_KNativePtr pinned;
} static_mytest_linux_amr64_kref_kotlin_Int;
typedef struct {
  static_mytest_linux_amr64_KNativePtr pinned;
} static_mytest_linux_amr64_kref_kotlin_Long;
typedef struct {
  static_mytest_linux_amr64_KNativePtr pinned;
} static_mytest_linux_amr64_kref_kotlin_Float;
typedef struct {
  static_mytest_linux_amr64_KNativePtr pinned;
} static_mytest_linux_amr64_kref_kotlin_Double;
typedef struct {
  static_mytest_linux_amr64_KNativePtr pinned;
} static_mytest_linux_amr64_kref_kotlin_Char;
typedef struct {
  static_mytest_linux_amr64_KNativePtr pinned;
} static_mytest_linux_amr64_kref_kotlin_Boolean;
typedef struct {
  static_mytest_linux_amr64_KNativePtr pinned;
} static_mytest_linux_amr64_kref_kotlin_Unit;
typedef struct {
  static_mytest_linux_amr64_KNativePtr pinned;
} static_mytest_linux_amr64_kref_kotlin_UByte;
typedef struct {
  static_mytest_linux_amr64_KNativePtr pinned;
} static_mytest_linux_amr64_kref_kotlin_UShort;
typedef struct {
  static_mytest_linux_amr64_KNativePtr pinned;
} static_mytest_linux_amr64_kref_kotlin_UInt;
typedef struct {
  static_mytest_linux_amr64_KNativePtr pinned;
} static_mytest_linux_amr64_kref_kotlin_ULong;
typedef struct {
  static_mytest_linux_amr64_KNativePtr pinned;
} static_mytest_linux_amr64_kref_MyTestClass;


typedef struct {
  /* Service functions. */
  void (*DisposeStablePointer)(static_mytest_linux_amr64_KNativePtr ptr);
  void (*DisposeString)(const char* string);
  static_mytest_linux_amr64_KBoolean (*IsInstance)(static_mytest_linux_amr64_KNativePtr ref, const static_mytest_linux_amr64_KType* type);
  static_mytest_linux_amr64_kref_kotlin_Byte (*createNullableByte)(static_mytest_linux_amr64_KByte);
  static_mytest_linux_amr64_KByte (*getNonNullValueOfByte)(static_mytest_linux_amr64_kref_kotlin_Byte);
  static_mytest_linux_amr64_kref_kotlin_Short (*createNullableShort)(static_mytest_linux_amr64_KShort);
  static_mytest_linux_amr64_KShort (*getNonNullValueOfShort)(static_mytest_linux_amr64_kref_kotlin_Short);
  static_mytest_linux_amr64_kref_kotlin_Int (*createNullableInt)(static_mytest_linux_amr64_KInt);
  static_mytest_linux_amr64_KInt (*getNonNullValueOfInt)(static_mytest_linux_amr64_kref_kotlin_Int);
  static_mytest_linux_amr64_kref_kotlin_Long (*createNullableLong)(static_mytest_linux_amr64_KLong);
  static_mytest_linux_amr64_KLong (*getNonNullValueOfLong)(static_mytest_linux_amr64_kref_kotlin_Long);
  static_mytest_linux_amr64_kref_kotlin_Float (*createNullableFloat)(static_mytest_linux_amr64_KFloat);
  static_mytest_linux_amr64_KFloat (*getNonNullValueOfFloat)(static_mytest_linux_amr64_kref_kotlin_Float);
  static_mytest_linux_amr64_kref_kotlin_Double (*createNullableDouble)(static_mytest_linux_amr64_KDouble);
  static_mytest_linux_amr64_KDouble (*getNonNullValueOfDouble)(static_mytest_linux_amr64_kref_kotlin_Double);
  static_mytest_linux_amr64_kref_kotlin_Char (*createNullableChar)(static_mytest_linux_amr64_KChar);
  static_mytest_linux_amr64_KChar (*getNonNullValueOfChar)(static_mytest_linux_amr64_kref_kotlin_Char);
  static_mytest_linux_amr64_kref_kotlin_Boolean (*createNullableBoolean)(static_mytest_linux_amr64_KBoolean);
  static_mytest_linux_amr64_KBoolean (*getNonNullValueOfBoolean)(static_mytest_linux_amr64_kref_kotlin_Boolean);
  static_mytest_linux_amr64_kref_kotlin_Unit (*createNullableUnit)(void);
  static_mytest_linux_amr64_kref_kotlin_UByte (*createNullableUByte)(static_mytest_linux_amr64_KUByte);
  static_mytest_linux_amr64_KUByte (*getNonNullValueOfUByte)(static_mytest_linux_amr64_kref_kotlin_UByte);
  static_mytest_linux_amr64_kref_kotlin_UShort (*createNullableUShort)(static_mytest_linux_amr64_KUShort);
  static_mytest_linux_amr64_KUShort (*getNonNullValueOfUShort)(static_mytest_linux_amr64_kref_kotlin_UShort);
  static_mytest_linux_amr64_kref_kotlin_UInt (*createNullableUInt)(static_mytest_linux_amr64_KUInt);
  static_mytest_linux_amr64_KUInt (*getNonNullValueOfUInt)(static_mytest_linux_amr64_kref_kotlin_UInt);
  static_mytest_linux_amr64_kref_kotlin_ULong (*createNullableULong)(static_mytest_linux_amr64_KULong);
  static_mytest_linux_amr64_KULong (*getNonNullValueOfULong)(static_mytest_linux_amr64_kref_kotlin_ULong);

  /* User functions. */
  struct {
    struct {
      struct {
        static_mytest_linux_amr64_KType* (*_type)(void);
        static_mytest_linux_amr64_kref_MyTestClass (*MyTestClass)();
        static_mytest_linux_amr64_kref_kotlin_Double (*get_testDouble)(static_mytest_linux_amr64_kref_MyTestClass thiz);
        void (*set_testDouble)(static_mytest_linux_amr64_kref_MyTestClass thiz, static_mytest_linux_amr64_kref_kotlin_Double set);
        static_mytest_linux_amr64_KInt (*get_testInt)(static_mytest_linux_amr64_kref_MyTestClass thiz);
        void (*set_testInt)(static_mytest_linux_amr64_kref_MyTestClass thiz, static_mytest_linux_amr64_KInt set);
      } MyTestClass;
      static_mytest_linux_amr64_KInt (*foo)(static_mytest_linux_amr64_KInt a, static_mytest_linux_amr64_KInt b);
    } root;
  } kotlin;
} static_mytest_linux_amr64_ExportedSymbols;
extern static_mytest_linux_amr64_ExportedSymbols* static_mytest_linux_amr64_symbols(void);
#ifdef __cplusplus
}  /* extern "C" */
#endif
#endif  /* KONAN_STATIC_MYTEST_LINUX_AMR64_H */
```

头文件中，定义了 Kotlin 中非空基本类型和可空基本类型与 C 语言中数据类型的映射关系，以及一个嵌套结构体，这个结构体提供了 Kotlin 源码中公开的各个 API ，以便 C 语言调用。

验证一下可以发现，对应关系如下：

- Kotlin 可空基本类型，对应 C 的包含指针的结构体
- Kotlin 非空基本类型，对应 C 的普通基本类型
- Kotlin 的类，对应 C 的包含指针的结构体
- Kotlin 的函数，对应 C 的函数指针


> 有兴趣可以看看 Kotlin 编译为静态库或动态库时生成头文件的源码：
> 
> https://github\.com/JetBrains/kotlin/blob/master/kotlin\-native/backend\.native/compiler/ir/backend\.native/src/org/jetbrains/kotlin/backend/konan/cexport/CAdapterApiExporter\.kt
> 


3. 编写 C 代码，引入头文件，并调用 Kotlin 代码的API：

```C
#include "static_mytest_linux_arm64_api.h"
#include <stdio.h>

int main() {
    // 创建 static_mytest_linux_arm64_ExportedSymbols 结构指针
    static_mytest_linux_arm64_ExportedSymbols *symbols = static_mytest_linux_arm64_symbols();
    // 调用 foo 函数
    printf("call foo result: %d\n", symbols->kotlin.root.foo(3, 5));

    // 创建 MyTestClass 对象
    static_mytest_linux_arm64_kref_MyTestClass obj = symbols->kotlin.root.MyTestClass.MyTestClass();
    
    // 修改 MyTestClass 对象的 testInt 属性值
    symbols->kotlin.root.MyTestClass.set_testInt(obj, 100);
    // 打印修改后的 MyTestClass 对象的 testInt 属性值
    printf("get testInt: %d\n", symbols->kotlin.root.MyTestClass.get_testInt(obj));
    
    // 修改 MyTestClass 对象的 testDouble 属性值
    symbols->kotlin.root.MyTestClass.set_testDouble(obj, symbols->createNullableDouble(22.2));
    // 获取 testDouble 的值
    static_mytest_linux_arm64_kref_kotlin_Double kDoubleRef = symbols->kotlin.root.MyTestClass.get_testDouble(obj);
    // 因为 testDouble 是可空的Double类型，所以建议判空
    if (kDoubleRef.pinned == NULL) {
        printf("testDouble is null\n");
    } else {
        printf("get testDouble: %f\n", symbols->getNonNullValueOfDouble(kDoubleRef));
    }

    return 0;
}
```

> 头文件中API的用法：https://kotlinlang\.org/docs/native\-dynamic\-libraries\.html\#use\-generated\-headers\-from\-c
> 
> 



4. 在 Linux 中编译和静态链接

```c++
clang mytest.c libstatic_mytest_linux_arm64.a  -lstdc++ -o mytest
```



5. 运行可执行文件

```Plain Text
./mytest

// 打印结果如下：
call foo result: 8
get init int: 0
get new int: 100
```



## 原理浅析

**要实现任何不同编程语言之间的交互，本质上都是要让它们在源码编译到可执行文件之间的某一个层次的类型是一样的**。

比如对于 Java 虚拟机来说，它只认识 Java 字节码，并不知道 Java 和 Kotlin 源代码的概念，所以 Java 和 Kotlin 的源代码都编译为 Java 字节码后，运行在 Java 虚拟机上就可以相互调用。

以 Linux 操作系统为例，对于链接器来说，它只知道 ELF 文件，并没有 C 源文件或者 Kotlin 源文件的概念，Kotlin 源码生成的 ELF 文件和 C 源码生成的 ELF 文件没有区别，所以把 Kotlin 编译为`.o`或者`.so`的 ELF 文件，自然就可以实现 C 语言调用 Kotlin 代码了。


下面来分析一下最终生成的可执行文件，看看 Kotlin 源码编译后的内容是什么：


先把`mytest`可执行文件的**代码段**数据打印到`mytest_code.txt`中（内容太多，如果直接打印在终端无法得到完整内容，所以重定向到文件）：

```Kotlin
objdump -d mytest > mytest_code.txt
```

查看`mytest_code.txt`文件内代码段中的`main`函数

![Image](./引用图片/Kotlin%20Native/Kotlin%20Native汇编展示1.png)

![Image](./引用图片/Kotlin%20Native/Kotlin%20Native汇编展示2.png)


在`main`函数中，我们调用了`static_mytest_linux_arm64_symbols`函数，调用它的指令解释如下：

```Kotlin
1ca70:        9400b90b         bl        4ae9c <static_mytest_linux_arm64_symbols> // 调用函数
1ca74:        f9000be0         str        x0, [sp, #16] // 保存返回值
1ca78:        f9400be8         ldr        x8, [sp, #16] // 加载返回值
1ca7c:        f9408908         ldr        x8, [x8, #272] // 加载函数指针到x8寄存器，偏移272也就是 0x110 
1ca80:        52800060         mov        w0, #0x3       // #3 调用 foo 的参数
1ca84:        528000a1         mov        w1, #0x5       // #5 调用 foo 的参数
1ca88:        d63f0100         blr        x8 // 取 x8 地址内的值指向的地址
```

![Image](./引用图片/Kotlin%20Native/Kotlin%20Native汇编展示3.png)

调用`static_mytest_linux_arm64_symbols`函数，返回值为0x80a00，加上 272（16进制为0x110），得到 0x80b10，放到 x8 寄存器，所以就是取0x80b10内的地址。



查看 \.data 数据段的数据：

```Kotlin
objdump -S --section=.data test > data.txt
```

![Image](./引用图片/Kotlin%20Native/Kotlin%20Native汇编展示4.png)

头文件中的`static_mytest_linux_arm64_ExportedSymbols`结构体提供的各个函数的地址，都会放在这里。0x80b10的值为 0x045914，所以会跳到这个位置执行指令。


![Image](./引用图片/Kotlin%20Native/Kotlin%20Native汇编展示5.png)

```Assembly language
这里面的核心代码：

// 把两个参数放到 w0 和 w1 寄存器
45158:        b9401fa0         ldr        w0, [x29, #28]
4515c:        b9401ba1         ldr        w1, [x29, #24]

// 调用 _konan_function_0 函数
45160:        97ffb9d4         bl        338b0 <_konan_function_0>
```


这个函数就是kotlin中的 foo 函数编译后的 native 指令

![Image](./引用图片/Kotlin%20Native/Kotlin%20Native汇编展示6.png)

```Assembly language
核心代码：

// 把 w0 和 w1 里面的参数保存到 w19 和 w20 寄存器
   33b10:        2a0103f3         mov        w19, w1
   33b14:        2a0003f4         mov        w20, w0
......
   
 // 把 w19 和 w20 寄存器的值相加，结果存储在 w0 寄存器中，作为函数的返回值
   33b20:        0b140260         add        w0, w19, w20
......
   33b2c:        d65f03c0         ret
```

这里就是 C 代码调用 Kotlin 中  生成 foo 函数的过程。



main 函数的后半部分就是使用 MyTestClass 相关的方法，同样是使用 0x80a00 加上偏移找到调用的指令地址。

```Assembly language
1ca9c:        f9400be8         ldr        x8, [sp, #16]
1caa0:        f9407508         ldr        x8, [x8, #232] // 0xe8
1caa4:        d63f0100         blr        x8
1caa8:        f90007e0         str        x0, [sp, #8]
1caac:        f9400be8         ldr        x8, [sp, #16]
1cab0:        f9408108         ldr        x8, [x8, #256] // 0x100
1cab4:        f94007e0         ldr        x0, [sp, #8]
1cab8:        d63f0100         blr        x8
```

比如这部分会使用 0x80a00 \+ 0xe8 （即0x80ae8）和 0x80a00 \+ 0x100（即0x80b00）的地址内的数据指向的指令。

指令就是`MyTestClass`相关的调用内容，调用流程的追踪方式和前面的分析过程一样，这里不再赘述，只需要了解一下的是，和类相关的这些语法特性，会涉及到很多 Kotlin Runtime API的调用，比如下图中调用的函数，从命名就可以看出是和类、对象这些概念相关的：

![Image](./引用图片/Kotlin%20Native/Kotlin%20Native汇编展示7.png)


### 总结

Kotlin 编译为 Native  产物，比如 Linux 上的 `.o`可重定位文件、`.so`动态库文件，和 C/C\+\+ 等语言编译的产物是一样的，所以 C 源码自然可以调用 Kotlin Native 产物。Kotlin 源码中公开的 API，都会转换为 C 中存在的语法， 放在生成的头文件中。


Kotlin Native 和 C\+\+/Objective\-C 一样，都有一个 Runtime 来支持复杂的语法特性，所以 Kotlin 代码生成的指令中会插入很多 Runtime API 的调用，如果不了解 Kotlin Runtime 源码的设计，这些指令的可读性就不太好，C\+\+ 和 Objective\-C 也是如此，而 C 语言生成的指令就非常容易和源码对应。

Kotlin Runtime 的源码在 https://github\.com/JetBrains/kotlin/tree/master/kotlin\-native/runtime/src 可以查看，都是由 C\+\+ 代码编写。



# Kotlin调用C的方式

C 语言调用 Kotlin Native 产物的情况中，Kotlin Native 产物提供了API（头文件），可以让 C 直接调用。反过来使用 Kotlin 调用 C 的话，也需要根据 C 公开的函数/变量生成 Kotlin 源码可以调用的产物，这个过程会使用到 Kotlin Native 编译工具中提供的 `cinterop`，下面将介绍它的使用流程。（实现原理和 klib 内容有关，所以会在介绍 klib 文件内容的时候分析 Kotlin 调用 C 的实现原理）



## 使用流程

`cinterop`是 Kotlin Native 提供的工具之一，它会根据 C 文件内的函数和全局变量生成`klib`，`klib`中包含了 C 函数和全局变量对应的 Kotlin 绑定，Kotlin 源码可以通过`klib`中的 Kotlin 绑定来调用 C 的函数和全局变量。（在下文的【klib文件内容】部分会简要分析原理）


下面来实际使用一下：


1. 编写一个 C 文件`mylib.h`，包含一个全局变量和一个函数：

```C
int my_global_var = 10;

int my_add(int a, int b){
    return a + b;
}
```

可以直接把定义和声明都写在这个头文件里面，对于简单的demo更方便



2. 编写一个`mylib.def`文件：

```Plain Text
headers = mylib.h

// 给C编译器设置的编译参数，这里设置头文件的include路径，否则headers就要设置全路径
compilerOpts = -I/Users/xxx/Desktop/kotlin_native
```

执行`cinterop`命令需要一个`.def`文件，我们可以通过这个`.def`文件中配置头文件、编译参数、链接参数、klib包名等，完整的选项可以在官方文档中查看：https://kotlinlang\.org/docs/native\-definition\-file\.html\#properties


> 通过`cinterop -h`查看命令选项，可以看到 def 文件里面支持的配置基本上都可以在命令行中设置，所以可以选择把某些配置写到 def 文件或者在命令行中配置，甚至可以全部在命令行中配置，def 文件为空也是可以成功执行的。但验证发现 \.def文件是必须存在的，即使 def 文件内容是空的。



3. 执行`cinterop`命令生成`klib`：

```C
cinterop -def ./mylib.def -o mylib
```

将生成`mylib.klib`



4. 在 Kotlin 源码中使用：

编写一个`TestCinterop.kt`文件，调用`klib`

```Java
import mylib.my_add
import mylib.my_global_var

@kotlinx.cinterop.ExperimentalForeignApi
fun main(){
    println(my_add(1, 2)) // 调用 C 中的 my_add 函数，打印结果
    println(my_global_var) // 获取 C 中的 my_global_var 变量，打印
    my_global_var = 20; // 修改  C 中的 my_global_var 变量
    println(my_global_var) // 再次获取 C 中的 my_global_var 变量，打印新的值
}
```



5. 使用 kotlinc\-native 编译，此时需要使用`-library`来设置依赖的 klib：

```Java
kotlinc-native ./TestCinterop.kt -library ./nativelib.klib -o testCinterop
```

在 macOS 或 Linux 上将生成`testCinterop.kexe`可执行文件。



总体来说，就是`cinterop`根据 C 代码的内容，生成了`klib`文件，Kotlin 源码可和`klib`文件一起编译生成可执行文件，也就实现了 Kotlin 调用 C。


## 要点介绍

上面介绍的使用流程只是一个简单的场景，比如直接把 C 代码中的声明和实现都写在一个文件了，而实际开发中一般会把需要公开的 API 放在头文件中，实现代码则会放在其他 C 文件中。所以这里会详细介绍一下`cinterop`和头文件的关系，和声明与实现分离情况的做法。


### cinterop和头文件

虽然官方文档中说`cinterop`需要和 C 的头文件配合使用，根据头文件来生成`klib`，但我们知道 C 的源文件和头文件只是概念上的约定，本身并没有任何区别。


实际上，`cinterop`会读取 C 文件中声明的函数、全局变量等信息，C文件的后缀扩展名是`.c`还是`.h`甚至是`.txt`都没有关系，只要内容是合法的 C 代码即可。假如 C 代码中的函数既有声明也有定义，那么生成的`klib`中就会包含声明和定义的全部代码信息（LLVM bitcode形式），全局变量同理；如果只有声明，没有定义，则`klib`就只包含声明的代码信息（同样是LLVM bitcode形式），但这种情况肯定就需要在链接时提供包含定义的代码。


### 提供C代码的实现部分

上面介绍使用流程的时候，就把声明和定义都写在 C 文件中了，所以不需要额外使用其他方式来提供实现代码。但实际稍微复杂的开发场景中，大概还是会使用声明和定义分离的方式，这种情况下，就需要用某种方式提供定义，否则显然会编译失败。



以下是一些提供实现代码的方式介绍，可以根据情况选择适合的方式：



1. **方式1**：在 def 文件中可以直接写 C 代码：

`test.h`内容如下：

```C
int getInt();
```



因为在`def`文件可以写 C 代码，那么头文件的实现代码也可以写在这里`test.def`，需要使用`---`分隔配置和代码，代码写在`---`符号之后：

```C
headers = test.h
compilerOpts = -I/Users/xxx/xxx
---

int getInt() {
    return 123;
}

// 还可以根据需要，继续写一些其他代码，不一定是和头文件定义相关的
float getFloat(){
    return 3.14;
}
```

然后就是同样的写 Kotlin 代码和编译流程。



> `cinterop`会把 def 文件中 \-\-\- 符号后的 C 代码和引用的头文件的内容一并处理，这里可以写任意的C代码，那么也就能用来写头文件的实现代码。
> 


2. **方式2**：在 def 的 headers 对应的 C 文件（或者说头文件）中直接写实现代码

和我们介绍使用流程的 demo 一样，直接把实现代码写到头文件里面



3. **方式3**：使用`def`文件的`linkerOpts`配置

`linkerOpts`配置可以用于设置 kotlinc\-native 编译时要链接的文件，可以通过它把头文件的实现部分链接起来。



`test.h`：

```C
int getNumber();
```

`test.c`：

```C
#include "test.h"

int getNumber() {
    return 123;
}
```

`test.c` 文件可以编译为`.o`可重定位文件，`.o`可以打包为`.a`静态库文件，也可以编译为`.so`或`.dylib`动态库文件。假设文件名叫`test.o`、`libtest.a` 、`libtest.so`，这三种形式都可以用于`linkerOpts`方式。



`test.def`文件：

```Java
headers = test.h
compilerOpts = -I/Users/xxx/Desktop/kotlin_native

// 1. 如果想链接 libtest.a 或者 libtest.so，使用 -l 选项
linkerOpts = -L/Users/xxx/Desktop/kotlin_native -ltest

// 2. 如果想链接 test.o 可重定位文件
linkerOpts = -L/Users/xxx/Desktop/kotlin_native test.o
```

同样使用`cinterop`命令生成`klib`

```C
cinterop -def test.def
```


`test.kt`文件：

```C
import test.getNumber

@kotlinx.cinterop.ExperimentalForeignApi
fun main(){
    println(getNumber())
}
```


执行编译：

```C
kotlinc-native test.kt -library ./nativelib.klib -o myProgram
```

将生成`myProgram.kexe`



4. **方式4**：使用 kotlinc\-native  命令的`-linker-option`选项

这个选项其实就是`def`文件中`linkerOpts`配置的命令行配置版本，使用这个方式的话，`def`文件就不需要使用

`linkerOpts`配置，而是在使用`kotlinc-native` 的时候，通过`-linker-option`配置`.o`、`.a`、`.so`文件。


比如链接静态库`libtest.a`：

```C
kotlinc-native test.kt -library ./nativelib.klib -linker-option libtest.a -o myProgram
```


5. **方式5**：官方文档中还提供了一个实现性的配置 `staticLibraries`和`libraryPaths`，它和使用`linkerOpts`的区别是，这个配置只支持静态库，并且会直接把静态库放到生成的`klib`文件内

```Java
headers = mylib.h
compilerOpts = -I/Users/huangfeiyang/Desktop/kotlin_native
staticLibraries = libmylib.a
libraryPaths = /Users/huangfeiyang/Desktop/kotlin_native
```

然后在使用 kotlinc\-native 编译链接时，就不需要该静态库了，因为已经存在于`klib`内：

```C
kotlinc-native test.kt -library ./nativelib.klib -o myProgram
```


6. **方式6**：使用 kotlinc\-native 命令的 `-native-library` 选项

这个选项可以在编译 Kotlin 时，包含 LLVM bitcode 形式的库。那么如果我们把 test\.c 编译为bitcode文件（\.bc），同样就可以实现链接。


把 C 文件编译为 LLVM bitcode：

```C
clang -emit-llvm test.c -c -o test.bc
```


编译时使用`-native-library`包含 C 代码生成的 bitcode：

```Java
kotlinc-native  test.kt -library ./nativelib.klib -native-library test.bc
```


> 以上介绍了通过实践了解到的几种方式，应该能覆盖常见的很多使用场景，官方文档没有详细介绍这方面的所有用法，如果以上方式不能满足，可以通过`cinterop -h`、`kotlinc-native -h` 查看命令的所有选项和`def`文件的所有配置，以了解是否还有其他方式可以使用。


> Kotlin 多平台的插件没有实现像 Android 中 NDK 开发一样把 C 源码放在项目内并在构建项目时自动完成 C 源码的编译链接工作，所以如果想在项目中让 Kotlin 调用的 C 以源码形式存在，就需要我们手动完成 C 源码的编译链接，如果想实现自动化完成，可以自己编写 Gradle 插件或者Task来完成这个事情。
> 
> 比如有一个Gradle插件项目：https://github\.com/touchlab/cklib，它会在编译Kotlin的任务之前插入另一个任务，该任务会把C代码编译为bitcode，然后利用`-native-library`完成和 Kotlin 的链接。
> 
> 也可以参考这个项目：https://github\.com/avan1235/kotlin\-native\-rust\-interop，它实现了 kotlin 调用 rust，利用了`linkerOpts`配置。
> 
> 
> 
> 但既然 Kotlin 可以编译为 Native，其实代码就可以都用 Kotlin 来写，比如很多库是 C 语言写的，我们使用它们编译产生的静态库/动态库就行了，并不需要 C 源码，可能这也是 Kotlin 多平台插件没有支持在 Kotlin 多平台项目中写 C 源码并自动完成编译链接工作的原因。但仍然会有一些情况，想把 C 源码放在 Kotlin 多平台项目中一起构建，比如个人喜好，或者使用 C 的开源库，但自己会做一些改动，把 C 源码也放到 Kotlin 多平台项目里面来更好管理，这些情况就可以参考上面介绍的两个项目的做法。
> 


## 映射关系

Kotlin 调用 C 代码时，由于语法差异，很多操作需要特定的转换。比如 Kotlin 语法并不能创建一个 C 的函数指针，所以 Kotlin Native 的运行库提供了`staticCFunction`函数，可以用它在 Kotlin 代码中创建 C 的函数指针：

```Kotlin
@ExperimentalForeignApi
@TypedIntrinsic(IntrinsicType.INTEROP_STATIC_C_FUNCTION)
public external fun <P1, R> staticCFunction(@VolatileLambda function: (P1) -> R): CPointer<CFunction<(P1) -> R>>
```


还有其他比如结构体、字符串等映射的函数，这部分官方文档写得比较清楚，直接查看即可：

https://kotlinlang\.org/docs/mapping\-primitive\-data\-types\-from\-c\.html

https://kotlinlang\.org/docs/mapping\-struct\-union\-types\-from\-c\.html

https://kotlinlang\.org/docs/mapping\-function\-pointers\-from\-c\.html

https://kotlinlang\.org/docs/mapping\-strings\-from\-c\.html



# klib文件内容

`klib`是 Kotlin 定义的二进制库文件格式，与Java的Jar文件类似，它本质上也是一个Zip文件。使用`kotlinc-native`可以把 Kotlin 源码编译为`klib`，使用 `cinterop`根据 C 头文件生成 Kotlin 的绑定时，也是生成`klib`的形式。

我们对这两种情况分别分析，从而具体了解 klib 的设计。

## cinterop生成的klib

前面看到使用 cinterop 可以根据 C 的函数声明创建一个 klib，kotlin代码可以通过这个 klib 调用 C 函数。


我们现在再写一个 C 的头文件，用它配合 def 文件和 cinterop 命令来生成 klib：

```Java
// mylib.h，函数名称用一个好区分的名称，方面后面分析的时候可以快速搜索到它

int test_kn_add(int a, int b);
```

生成的`klib`只是一个zip文件，我们直接解压看它的内容：

![Image](./引用图片/Kotlin%20Native/klib文件内容.png)

这里面只有4个有效文件，其他都是空目录，下面依次分析这几个文件的内容。


> cinterop 使用 staticLibraries 和 libraryPaths 的情况，就会把静态库文件放到 targets/macos\_arm64（或者其他平台目录）/included/ 目录里面
> 


### module

内容只是模块名称信息，看起来不是跨语言调用的关键

![Image](./引用图片/Kotlin%20Native/klib%20module文件内容.png)



### manifest

清单文件里面，就是一些版本、平台、包名等信息的描述

```Java
abi_version=1.201.0
builtins_platform=NATIVE
compilerOpts=-I/Users/huangfeiyang/Desktop/kotlin_native/cinteropklib
compiler_version=2.1.10
depends=stdlib
exportForwardDeclarations=
headers=mylib.h
includedForwardDeclarations=
includedHeaders=b63b9e92c9d52e3de55132c69b7323fe884d1d60fb3ad6611231003ea1b7ffd4 0e1b43fc5478a5d1929304ed3ef63689167d8724be7b0adf69765ba194489ba8
interop=true
ir_provider=kotlin.native.cinterop
ir_signature_versions=1,2
metadata_version=1.4.1
native_targets=macos_arm64
package=mylib
unique_name=nativelib

```



### 0\_mylib\.knm

knm 文件是 klib 的元数据文件，在 kotlin github 项目的源码中可以搜到它的命名：

```Kotlin
// kotlin语言官方源码中的常量，可以看出 knm 文件就是klib的元数据文件
const val KLIB_METADATA_FILE_EXTENSION = "knm"
```



通过 Android Studio 反编译查看，可以看到就是一个 Kotlin 函数信息，刚好对应 C 头文件中的声明函数：

![Image](./引用图片/Kotlin%20Native/knm反编译信息.png)

\.knm 文件提供了`klib`模块对外公开的 Kotlin 函数、变量等声明信息，如果在 Kotlin源码中调用`test_kn_add`函数，并和这个 klib 文件一起编译，Kotlin Native 编译器就可以通过 klib 中的 knm 文件内容知道 klib 提供的函数等情况，完成 Kotlin 源码和 klib 的编译链接。



函数上使用了`@kotlinx.cinterop.internal.CCall`注解，它的源码如下：

```kotlin
public annotation class CCall(val id: String)
```

反编译的结果并不能看到这个`id`参数是什么，所以再直接查看16进制文本：

![Image](./引用图片/Kotlin%20Native/knm%2016进制信息.png)

通过查看文本，可以看到是`id`是`knifunptr_mylib0_test_kn_add`，它在`cstubs.bc`中会用到。


### cstubs\.bc

`cstubs.bc`是LLVM Bitcode文件，是LLVM IR的持久化形式，可以把它转换为可读性更好的格式来查看：

```Kotlin
// 通过命令，将 bc 文件转为 ll 文件
clang -emit-llvm -S cstubs.bc -o cstubs.ll
```



`cstubs.ll`文件内容如下：

```Java
; ModuleID = '/Users/huangfeiyang/Desktop/kotlin_native/default/targets/macos_arm64/native/cstubs.bc'
source_filename = "-"
target datalayout = "e-m:o-i64:64-i128:128-n32:64-S128"
target triple = "arm64-apple-macosx15.0.0"

@"\01knifunptr_mylib0_test_kn_add" = local_unnamed_addr global ptr @mylib_test_kn_add_wrapper0, align 8

; Function Attrs: alwaysinline uwtable(sync)
define i32 @mylib_test_kn_add_wrapper0(i32 noundef %0, i32 noundef %1) #0 {
  %3 = tail call i32 @test_kn_add(i32 noundef %0, i32 noundef %1)
  ret i32 %3
}

declare i32 @test_kn_add(i32 noundef, i32 noundef) local_unnamed_addr #1

attributes #0 = { alwaysinline uwtable(sync) "frame-pointer"="non-leaf" "min-legal-vector-width"="0" "no-trapping-math"="true" "stack-protector-buffer-size"="8" "target-cpu"="apple-m1" "target-features"="+aes,+crc,+crypto,+dotprod,+fp-armv8,+fp16fml,+fullfp16,+lse,+neon,+ras,+rcpc,+rdm,+sha2,+sha3,+sm4,+v8.1a,+v8.2a,+v8.3a,+v8.4a,+v8.5a,+v8a,+zcm,+zcz" }
attributes #1 = { "frame-pointer"="non-leaf" "no-trapping-math"="true" "stack-protector-buffer-size"="8" "target-cpu"="apple-m1" "target-features"="+aes,+crc,+crypto,+dotprod,+fp-armv8,+fp16fml,+fullfp16,+lse,+neon,+ras,+rcpc,+rdm,+sha2,+sha3,+sm4,+v8.1a,+v8.2a,+v8.3a,+v8.4a,+v8.5a,+v8a,+zcm,+zcz" }

!llvm.module.flags = !{!0, !1, !2, !3, !4}
!llvm.ident = !{!5}

!0 = !{i32 2, !"SDK Version", [2 x i32] [i32 15, i32 2]}
!1 = !{i32 1, !"wchar_size", i32 4}
!2 = !{i32 8, !"PIC Level", i32 2}
!3 = !{i32 7, !"uwtable", i32 1}
!4 = !{i32 7, !"frame-pointer", i32 1}
!5 = !{!"clang version 16.0.0 (https://github.com/Kotlin/llvm-project 4e2cc6a8ac2afae0d0ddde4a51818642f0f5304f)"}

```

`knm`文件中的`@CCall`注解设置的`knifunptr_mylib0_test_kn_add`出现在这里，它是一个全局变量，赋值为函数地址，前面的@代表是全局的，\\01是要求名称在编译过程中保持不被改变。命名格式 `knifunptr_` 是Kotlin Native的命名约定



`knifunptr_mylib0_test_kn_add`对应 `mylib_test_kn_add_wrapper0`函数，该函数内调用了`test_kn_add`函数，`test_kn_add`函数在这里面只有声明没有定义。



也可以把 LLVM Bitcode 文件编译为可重定位文件，通过 ELF 文件的形式来查看：

```Kotlin
clang -c cstubs.bc -o cstubs.o
```

cstubs\.o 的符号表和代码段内容如下：

![Image](./引用图片/Kotlin%20Native/cstubs%20elf信息.png)



所以可以看出来 Kotlin 调用 klib 的 knm 中提供的 `test_kn_add` 函数，编译器会根据 klib 中 knm 文件中@CCall注解 id 找到 cstubs 中的同名全局变量，全局变量的值为`mylib_test_kn_add_wrapper0`函数的地址，从而通过此函数调用到 C 代码中的 `test_kn_add`函数，从而实现了 Kotlin 调用 C。

![Image](./引用图片/Kotlin%20Native/Kotlin%20Native内部调用原理.png)

> 如果 cinterop 使用的 c 头文件或者 def 文件内已经实现了函数，那么生成的 klib 内的 cstubs\.bc 也会有包含实现
> 
> 



**验证上述结论：**

编写一个实现了 test\_kn\_add 函数的 C文件 **mylib\.c**：

```C
int test_kn_add(int a, int b) {
    return a + b;
}
```

并编译为可重定位文件：

```C
clang -c mylib.c -o mylib.o
```


因为我们要在 Linux ARM64 平台验证，所以使用cinterop生成 klib 时，要指定目标平台：

```C
cinterop -target linux_arm64 -def mylib.def
```


在编写一个 Kotlin 文件，调用 klib 提供的 C 函数：

```C
@kotlinx.cinterop.ExperimentalForeignApi
fun main(){
    val result = mylib.test_kn_add(1, 2)
    println("result: $result")
}
```

完成编译链接，生成 ELF 可执行文件：

```C
kotlinc-native test.kt -target linux_arm64 -library nativelib.klib -linker-option mylib.o
```

分别获取可执行文件的代码段和数据段内容：

```C
objdump -d program.kexe  > code.txt
objdump -S --section=.data program.kexe > data.txt
```

![Image](./引用图片/Kotlin%20Native/Kotlin%20Native汇编展示8.png)

这里就是 Kotlin 代码调用 **mylib\.test\_kn\_add\(1, 2\) **跳转的指令

![Image](./引用图片/Kotlin%20Native/Kotlin%20Native汇编展示9.png)

2784的16进制为0xae0，所以这里会获取 0x2a7000 \+ 0xae0 = **0x2a7ae0** 的数据，并跳转该数据指向的地址。

![Image](./引用图片/Kotlin%20Native/Kotlin%20Native汇编展示10.png)

0x2a7ae0地址的数据就是全局变量`knifunptr_mylib0_test_kn_add`的值，即 **0x2620ac**


0x2620ac就是`mylib_test_kn_add_wrapper0`函数的地址，内部再调用`test_kn_add`函数：

![Image](./引用图片/Kotlin%20Native/Kotlin%20Native汇编展示11.png)

![Image](./引用图片/Kotlin%20Native/Kotlin%20Native汇编展示12.png)

以上就是最终的可执行文件中，在机器指令级别 Kolin 代码调用 C 代码的流程。



**总的来说，因为 Kotlin 会被编译为 LLVM IR，cstubs\.bc 也是 LLVM IR，Kotlin Naitve 编译器通过 knm 文件内的声明信息，把 Kotlin 源码的 LLVM 和 cstubs\.bc 的 LLVM IR 对接起来一起编译，这就是 cinterop 工具生成 klib 实现 Kotlin 源码调用 C 的方式。**



## Kotlin源码编译的klib

使用 kotlinc\-native 将 Kotlin 源码编译为 klib，它的内容和 cinterop 生成的 klib 有所不同。这种场景的 klib 用于在 Kotlin/Native 项目中共享和重用代码，包含了编译后的 Kotlin IR 和元数据。



编写一个`bar.kt`：

```Kotlin
fun test_bar(a: Int, b: Int): Int {
    return a + b
}
```

将它编译为 klib：

```Kotlin
kotlinc-native bar.kt -p library -o bar.klib
```



解压 bar\.klib 内容如下

![Image](./引用图片/Kotlin%20Native/klib文件内容2.png)

和 cinterop 生成的 klib 不一样，这种情况生成的 klib 会有 ir 目录，而 targets/macos\_arm64里面的目录都是空的，没有 cstubs\.bc 这种文件。


module文件和 cinterop 生成的klib一样，都是相关的信息描述。下面主要看menifest、knm 文件和 ir 目录的内容


### manifest

Kotlin 源码生成的 klib，manifest 文件内容的描述内容比 cinterop 生成的更少：

```xml
abi_version=1.201.0
builtins_platform=NATIVE
compiler_version=2.1.10
depends=stdlib
ir_signature_versions=1,2
metadata_version=1.4.1
native_targets=macos_arm64
unique_name=library

```



### Knm

通过 Android Studio 和 16 进制文本两种形式分别查看这里面的 knm 文件：

![Image](./引用图片/Kotlin%20Native/knm反编译信息2.png)

![Image](./引用图片/Kotlin%20Native/knm%2016进制信息2.png)

knm 文件中同样是 Kotlin 函数的声明信息，但不一样的是，这种情况的 knm 内的函数声明没有 `@CCall`注解，因为它就是简单的kotlin函数签名，并没有和 C 函数有绑定关系。



### ir 目录

回顾一下 Kotlin 的编译流程：Kotlin 代码首先被编译为 Kotlin IR，然后经过一系列优化后，转换为 LLVM IR，最后由 LLVM 生成目标平台的机器码。


ir 目录中存放了`knd`、`knf`、`knt`和`knb`几种扩展名的文件，它们就是kotlin IR 的序列化文件形式。`klib`作为 Kotlin 的二进制库文件格式，也要参与跨平台编译，所以这里面存放的是 Kotlin IR 这种中间形式，而不是最终的平台相关的机器码。


官方文档对 Kotlin IR 的介绍几乎没有，因为它是 Kotlin 编译器的内部实现细节，主要面向编译器开发者和高级用户，也就没有给普通开发者提供对 `knd`、`knf`、`knt`和`knb` 文件的逐个解析工具。但我们可以使用`klib`命令工具来查看klib模块中的整体 IR 信息：

```Plain Text
klib dump-ir bar.klib
```

打印结果如下：

```xml
MODULE_FRAGMENT name:<library>
  FILE fqName:<root> fileName:/Users/huangfeiyang/Desktop/kotlin_native/normalklib/bar.kt
    FUN name:test_bar visibility:public modality:FINAL <> (a:<unbound IrClassSymbolImpl>, b:<unbound IrClassSymbolImpl>) returnType:<unbound IrClassSymbolImpl>
      VALUE_PARAMETER name:a index:0 type:<unbound IrClassSymbolImpl>
      VALUE_PARAMETER name:b index:1 type:<unbound IrClassSymbolImpl>
      BLOCK_BODY
        RETURN type=<unbound IrClassSymbolImpl> from='public final fun test_bar (a: <unbound IrClassSymbolImpl>, b: <unbound IrClassSymbolImpl>): <unbound IrClassSymbolImpl> declared in <root>'
          CALL 'UNBOUND IrSimpleFunctionSymbolImpl' type=<unbound IrClassSymbolImpl> origin=PLUS
            $this: GET_VAR 'a: <unbound IrClassSymbolImpl> declared in <root>.test_bar' type=<unbound IrClassSymbolImpl> origin=null
            1: GET_VAR 'b: <unbound IrClassSymbolImpl> declared in <root>.test_bar' type=<unbound IrClassSymbolImpl> origin=null
```

虽然不了解 Kotlin IR 的语法，但是通过这里面的文字描述，也可以看得出来它就是`test_bar`函数。


> klib 命令工具还有其他一些命令选项，可以利用它了解 klib 的内容
> 
 

## 总结

`cinterop klib`与常规的`Kotlin klib`有所不同，`cinterop klib`只会包含 Kotlin 接口元数据，和 C 代码编译生成的`cstubs.bc`文件，但不包含 Kotlin IR；而`Kotlin klib`包含 Kotlin 接口元数据和 Kotlin IR，但没有`cstubs.bc`文件。它们的文件内容组成，也可以印证它们的不同用途。



大多数情况下，我们并不需要使用`cinterop`手动生成所有的 C 库绑定，因为 Kotlin Native SDK 已经提供了大部分平台的原生库绑定。例如：

- Linux POSIX

- Windows Win32

- macOS/iOS Apple Framework、POSIX

- 以及各平台的常用热门库，OpenGL， zlib 等



这些 Kotlin Native 官方提供的特定各种平台的klib库都放在`~/.konan/kotlin-native-prebuilt-macos-aarch64-2.1.10/klib/platform`目录里面。而 Kotlin Native 提供的 Kotlin 标准库则放在`～/.konan/kotlin-native-prebuilt-macos-aarch64-2.1.10/klib/common/stdlib`目录内。



参考资料：

klib文件格式介绍：https://kotlinlang\.org/docs/native\-libraries\.html\#library\-format

kotlin编译器源码中的klib文件内容定义：https://github\.com/JetBrains/kotlin/blob/master/compiler/util\-klib/src/org/jetbrains/kotlin/library/KotlinLibraryLayout\.kt

Kotlin 编译介绍：https://github\.com/JetBrains/kotlin/blob/master/docs/native/compilation\-model\.md



# 其他

## 和其他语言的交互⚒️

本文介绍 Kotlin Native 跨语言交互的场景都是使用 C 语言来分析，而 Kotlin Native 还支持和 Objective\-C、Swift 交互，可以查看文档：https://kotlinlang\.org/docs/native\-objc\-interop\.html、https://github\.com/kotlin\-hands\-on/kotlin\-swift\-interopedia



## 内存管理🧹

Kotlin\-Native 采用和 JVM 类似的方式，由另外的线程自动运行GC垃圾回收器利用根搜索算法回收内存，这样的做法，可以保持 Kotlin 在不同平台的一致用法，所以使用 Kotlin Native 不用像 C/C\+\+ 一样手动管理内存，也不用像 OC/Swift 采用的自动引用计算ARC一样需要处理循环引用。



但如果 Kotlin Native 代码和 C、OC、Swift 交互，那么还是需要了解这些语言的内存管理方式，比如一个 Swift 对象传到 Kotlin 函数中，或者 Kotlin 对象传到 Swift 函数中，内存回收将如何进行的问题。

可以查看官方文档中的内存管理介绍：

https://kotlinlang\.org/docs/native\-memory\-manager\.html

https://kotlinlang\.org/docs/native\-arc\-integration\.html



## KMP生态开源参考

https://github\.com/AAkira/Kotlin\-Multiplatform\-Libraries

https://github\.com/terrakok/kmp\-awesome

https://github\.com/joreilly

