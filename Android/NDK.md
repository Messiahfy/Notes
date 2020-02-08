NDK side by side为最新的

下面来看一下Android所提供的NDK根目录下的结构。

ndk-build：该Shell脚本是Android NDK构建系统的起始点，一般在项目中仅仅执行这一个命令就可以编译出对应的动态链接库了，后面会有详细的介绍。
ndk-gdb：该Shell脚本允许用GUN调试器调试Native代码，并且可以配置到Eclipse的IDE中，可以做到像调试Java代码一样调试Native的代码。
ndk-stack：该Shell脚本可以帮助分析Native代码崩溃时的堆栈信息，后续会针对Native代码的崩溃进行详细的分析。
build：该目录包含NDK构建系统的所有模块。
platforms：该目录包含支持不同Android目标版本的头文件和库文件，NDK构建系统会根据具体的配置来引用指定平台下的头文件和库文件。
toolchains：该目录包含目前NDK所支持的不同平台下的交叉编译器——ARM、x86、MIPS，其中比较常用的是ARM和x86。构建系统会根据具体的配置选择不同的交叉编译器。


[官方Android Studio配置ndk开发](https://developer.android.google.cn/studio/projects/add-native-code)

[官方ndk开发文档](https://developer.android.google.cn/ndk/guides/concepts)

[终于找到一篇极佳的 NDK 入门文章](https://mp.weixin.qq.com/s/Pg4pKhSScK8NgtT2RDw2uQ)