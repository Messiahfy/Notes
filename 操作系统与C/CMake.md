## 概述
C/C++的编译链接流程在项目较大的情况，直接使用GCC来操作是比较麻烦的，所以在各平台都产生了`make`这样的构建工具来管理编译链接等流程。但是使用`make`需要编写`Makefile`，`Makefile`描述了整个工程的编译、链接的规则。但是`Makefile`的格式在不同平台是不同的，所以又产生了`CMake`这样的工具，`CMake`使用平台无关的`CMakeList.txt`文件来自定义编译流程，`CMakeList.txt`会生成特定目标平台的`Makefile`相关文件。

在Android的NDK开发中，官方推荐我们使用CMake工具，也就是编写`CMakeList.txt`来自定义C/C++部分的编译链接流程。在Android Studio中，编写了`CMakeList.txt`后直接构建项目即可，Gradle会完成剩下的步骤。但是这里为了更清晰地了解`cmake`的使用，就以基本的使用为例，所以需要手动完成剩下的执行`cmake`和`make`命令。

cmake的核心使用流程：
1. 编写CMake的配置文件CMakeLists.txt
2. 执行`cmake path`，`path`为CMakeLists.txt文件所在的目录，将生成Makefile相关文件
3. 执行`make`命令编译

## 使用示例
### 单个编译模块
假设在当前目录有一个main.cpp文件，我们需要通过`cmake`来生成一个可执行文件。

项目目录结构：
```
dir/
├── CMakeLists.txt
├── main.cpp
```

1. 首先编写CMakeLists.txt文件
```
# CMake 最低版本号要求
cmake_minimum_required (VERSION 3.4.1)

# 项目信息
project (Test)

# 指定生成目标
add_executable(test main.cpp)
```
CMake的命令，不区分大小写。这三个命令的作用如下：
1. `cmake_minimum_required`：指定运行此配置文件所需的 CMake 的最低版本
2. `project`：参数值是 Test，该命令表示项目的名称是 Test
3. `add_executable`：将名为 main.cpp 的源文件编译成名为 test 的可执行文件

2. 然后执行cmake命令：
```
cmake .
```
这里因为就在当前目录执行，CMakeLists.txt就在当前目录，所以路径为.

3. 执行make命令
执行`cmake`后，将生成Makefile相关文件，然后执行`make`命令即可生成可执行文件

-----------

如果源文件有多个，则可以在add_executable命令中添加多个文件：
```
add_executable(test main.cpp a.cpp b.cpp)
```
如果源文件很多，可以包含整个目录：
```
# 查找当前目录下的所有源文件
# 并将名称保存到 DIR_SRCS 变量
aux_source_directory(. DIR_SRCS)

# 指定生成目标
add_executable(test ${DIR_SRCS})
```

-----------

如果不想生成可执行文件，而是生成链接库：
```
// 静态链接库（默认）
add_library(libname STATIC ${DIR_SRCS})

// 动态链接库
add_library(libname SHARED ${DIR_SRCS})
```

### 多个编译模块
我们可以把一个项目分为多个模块，在每个模块都写一个CMakeLists.txt，这里的示例，parent目录和sub目录分别为一个模块，sub目录是parent目录的子目录，main.cpp中要使用sub目录中的链接库。
```
./parent
    |
    +--- main.cpp
    |
    +--- sub/
          |
          +--- ...省略
```

首先编写父模块的CMakeLists.txt文件：
```
cmake_minimum_required (VERSION 3.4.1)

project (Parent)

aux_source_directory(. DIR_SRCS)

# 添加 sub 模块的目录
add_subdirectory(sub)

# 指定生成可执行文件
add_executable(test main.cpp)

# 添加链接库
target_link_libraries(test subLib)
```
1. `add_subdirectory`：添加一个子目录用于构建，目录可以是相对路径，也可以是绝对路径。注意：如果sub不是当前目录的子目录，那么add_subdirectory目录还需要设置第二个参数，用于指定sub模块的生成文件，比如库文件的放置路径

2. `target_link_libraries`：指定将生成的可执行文件test需要链接的库，这里这个库文件在子模块中生成

子模块的CMakeLists.txt文件：
```
aux_source_directory(. DIR_LIB_SRCS)

# 生成链接库
add_library (subLib ${DIR_LIB_SRCS})
```
子模块生成一个名为subLib的静态链接库，父模块通过`add_subdirectory`命令依赖子模块，然后通过`target_link_libraries`命令为将要生成的可执行文件test添加链接库subLib。

----------

如果要导入已存在的链接库：
```
add_library( imported-lib SHARED IMPORTED )
set_target_properties( imported-lib PROPERTIES IMPORTED_LOCATION imported-lib/src/${ANDROID_ABI}/libimported-lib.so)
```
导入imported-lib库，库的IMPORTED_LOCATION属性（也就是路径）的值为后面的imported-lib/src/${ANDROID_ABI}/libimported-lib.so

## 常用命令
还有很多其他命令，可以阅读[官方文档](https://cmake.org/cmake/help/v3.19/manual/cmake-commands.7.html)，下面是一些常见的例子：

### find_library
命令的签名为：
```
find_library (<VAR> name1 [path1 path2 ...])
```
第一个参数VAR将存储找到的name1库的路径，如果不传[path1 ...]，则在默认路径中搜索
```
find_library( # Sets the name of the path variable.
              log-lib

              # Specifies the name of the NDK library that
              # you want CMake to locate.
              log )
```
这个例子就是在默认路径中搜索log库，并把路径存到log-lib中

### include_directories
使用include_directories命令，可以添加我们要使用的头文件的路径，例如：
```
include_directories("src/headers/")
```