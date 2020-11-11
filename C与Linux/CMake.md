## 概述
CMake使用平台无关的 CMakeList.txt 文件来自定义编译流程，然后根据目标平台生成本地化的 Makefile 和工程文件。

使用流程：
1. 编写CMake的配置文件CMakeLists.txt
2. 执行`cmake path`，`path`为CMakeLists.txt文件所在的目录，将生成Makefile相关文件
3. 执行`make`命令编译

## 使用示例
### 单个编译模块
首先编写CMakeLists.txt文件
```
# CMake 最低版本号要求
cmake_minimum_required (VERSION 3.4.1)

# 项目信息
project (Test)

# 指定生成目标
add_executable(test main.cpp)
```
CMake的命令，不区分大小写。
1. `cmake_minimum_required`：指定运行此配置文件所需的 CMake 的最低版本
2. `project`：参数值是 Test，该命令表示项目的名称是 Test
3. `add_executable1`：将名为 main.cpp 的源文件编译成名为 test 的可执行文件

然后执行camke命令：
```
// 就在当前目录执行，则路径为.
cmake .
```
执行`cmake`后，将生成Makefile，然后执行`make`命令即可生成可执行文件


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
add_executable(Demo ${DIR_SRCS})
```


如果不想生成可执行文件，而是生成链接库：
```
// 静态链接库（默认）
add_library(libname STATIC ${DIR_SRCS})

// 动态链接库
add_library(libname SHARED ${DIR_SRCS})
```

### 多个编译模块
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

# 指定生成目标 
add_executable(test main.cpp)

# 添加链接库
target_link_libraries(test subLib)
```

子模块的CMakeLists.txt文件：
```
aux_source_directory(. DIR_LIB_SRCS)

# 生成链接库
add_library (subLib ${DIR_LIB_SRCS})
```
子模块生成一个名为subLib的静态链接库，父模块通过`add_subdirectory`命令依赖子模块，然后通过`target_link_libraries`命令为将要生成的可执行文件test添加链接库subLib。

如果要导入已存在的链接库：
```
add_library( imported-lib SHARED IMPORTED )
set_target_properties( imported-lib PROPERTIES IMPORTED_LOCATION imported-lib/src/${ANDROID_ABI}/libimported-lib.so)
```
导入imported-lib库，库的IMPORTED_LOCATION属性（也就是路径）的值为后面的imported-lib/src/${ANDROID_ABI}/libimported-lib.so

## 常用命令
常用命令可以阅读[官方文档](https://cmake.org/cmake/help/v3.19/manual/cmake-commands.7.html)

find_library命令的签名为：
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