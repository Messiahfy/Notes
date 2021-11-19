## 项目结构
一个软件项目，一般都会有添加依赖库、编译、打包等常见的流程，Android项目更是如此，为了方便完成这些工作，Gradle这类工具出现了。一个Android的Gradle项目的大致目录结构如下：
```
├── app
│   ├── build.gradle
│   ├── ...
│   └── src
│       ├── ...
├── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradle.properties
├── gradlew
├── gradlew.bat
├── local.properties
└── settings.gradle
```
* gradle-wrapper：为了实现多个开发者的Gradle版本一致，Gradle项目一般都会使用gradle-wrapper，版本在gradle-wrapper.properties配置，并使用`gradlew`来执行命令。
* settings.gradle：对应Gradle的Settings对象，用于设定项目结构，例如使用`include`添加子项目。
* build.gradle：Gradl构建脚本，Gradle项目可以存在父子关系，所以每个Gradle项目都可以有脚本文件，例如这里根目录和app目录都有build.gradle文件。在这个脚本文件中设定构建项目过程的各种信息，比如依赖库。
* gradle.properties：gradle、jvm、构建过程等各种配置。
* local.properties：本地配置，例如Android SDK路径。

## 构建流程
使用Gradle的时候，例如`./gradlew clean`，或者`gardle clean`（根据是否使用gradle-wrapper而定）。如果使用gradle-wrapper，那么首先会下载对应的gradle版本。然后经过下面三个阶段：
1. 初始化：Gradle支持多项目构建，对应Android的Project+多Module，这个阶段的核心就是执行`settings.gradle`，通过include之类的函数，Gradle就知道了当前项目里面哪些Project会参与构建。（还有执行init.gradle、解析gradle.properties、编译buildSrc目录等）
2. 配置：执行各个Project中的`build.gradle`文件，一般我们都会在`build.gradle`中创建各种Task，并可以使Task之间形成依赖关系，例如一个build任务需要前置执行多个任务（BuildListener可以设置构建过程的回调，例如开始和结束）。Android Studio创建的默认工程的`build.gradle`看不到明显的创建Task，以及提供了android的配置DSL，这些功能写在插件中，插件在后面介绍。
3. 执行：Task就是根据具体情况而定的功能代码，例如这里的`clean`任务就是简单地把编译构建的产物文件删掉。如果一个任务依赖其他任务，那么会先执行其他任务。

> 参考文档：https://docs.gradle.org/current/userguide/build_lifecycle.html

## 核心概念
### Project
Gradle构建中包含一个或多个project，project的功能取决于你编写的Gradle构建脚本，一般会在其中编写各种Task。Task的功能是任意的，例如编译代码、删除文件、构建apk。很多常用的Task并不需要我们自己写，可以使用Plugin，Plugin就是Gradle构建脚本，使用它相当于我们平时开发会去使用一些开源库。

每个project都对应一个build.gradle文件，也对应一个Gradle中的Project实例，在build.gradle中**直接调用**任何方法都是调用的Project实例的方法，也就相当于build.gradle中的代码是处于Project实例的作用域中。

部分方法（具体参考[Project文档](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html)）：
```
//插件相关，继承自PluginAware
apply​(Closure closure)
apply​(Map<String,​?> options)
apply​(Action<? super ObjectConfigurationAction> action)
getPluginManager()
getPlugins()

//获取ExtensionContainer，和自定义DSL相关
getExtensions()

//构建脚本自身配置
buildscript​(Closure configureClosure)

//依赖相关
dependencies​(Closure configureClosure)
getDependencies()
configurations​(Closure configureClosure)
getConfigurations()

//文件相关
file​(Object path)
fileTree​(Object baseDir)
delete​(Object... paths)
mkdir​(Object path)
...

//Task相关
task​(String name, Closure configureClosure)
...

//属性
getVersion()
getRootDir()
setGroup​(Object group)
...

//多项目管理
allprojects​(Closure configureClosure)
project​(String path)
subprojects​(Closure configureClosure)
...

//监听
afterEvaluate​(Closure closure)
beforeEvaluate​(Closure closure)
...

//其他具体的到文档中查看
```
> project路径的分隔符是 : 而不是 /，可以在settings.gradle或者执行某个Task时，看到使用 :。

### Task
因为Gradle在配置过程会执行`build.gradle`文件，所以我们可以在`build.gradle`中使用创建Task，然后才能使用`./gradlew xxx`来执行Task。
```
//创建Task有多种方式，这里任选一种，并且明确使用project来调用，更清晰一点
project.task("MyTask"){
    println("在配置阶段打印")
    doFirst {
        println("在执行阶段打印B")
    }
    doFirst {
        println("在执行阶段打印A")
    }
    doLast {
        println("在执行阶段打印C")
    }
}
```
创建Task，传入的闭包是在配置阶段执行，在闭包内的`doFirst`这类方法，是给Task添加Action，Action会在Task的执行阶段执行。一个Task可以有多个Action。`doFirst`和`doLast`都是把Action添加到Task的Action列表中，`doFirst`是插入到列表头部，`doLast`是插入到列表尾部，都是可以多次调用的。

> 还可以使用定义类的方式，结合注解，来创建Task，具体查看[官方文档](https://docs.gradle.org/current/userguide/more_about_tasks.html)。

常用方法（[详细文档](https://docs.gradle.org/current/dsl/org.gradle.api.Task.html)）：
```
//task依赖
dependsOn(Object... paths)

//动作定义
doFirst(Closure action)
doLast(Closure action)
getActions()

//输入/输出数据声明
getInputs()
getOutPuts()

//getter/setter属性
getAnt()
getDescription()
getEnabled()
getGroup()
setDescription(String description)
setEnabled(boolean enabled)
setGroup(String group)
```

#### Task依赖关系
Task还可以设置依赖关系和执行顺序，使用`dependsOn`、`mustRunAfter`等方法。
```
task0.dependsOn startSession
task2.dependsOn task1, task0
```

#### Task语法糖
Task还有多种创建方式：
```
project.tasks.create("myTask").doLast {
    println "xxxxxx"
}

task copyDocs(type: Copy) {
    from 'src/main/doc'
    into 'build/target/doc'
}

task  helloWorld {
    doLast {
          println 'Hello world'
    }
}

//可以使用<<代替doLast
task YourTask << {
    println "Hello"
}
```
重点看一下上面`helloWorld`这个task的创建方式：
```
//方法声明
Task task(String name, Closure configureClosure)

//完整调用
task("helloWorld", {
    doLast {
          println 'Hello world'
    }
})

//Groovy语法糖简写
task "helloWorld" {
    doLast {
          println 'Hello world'
    }
}

//Gradle中的简写：
task  helloWorld {
    doLast {
          println 'Hello world'
    }
}
```
在Gradle中，甚至可以去掉字符串双引号，这不符合Groovyy语法，那么为什么可以这样写，原因在于Gradle会自己增加一些语法转换，在Groovy语法之外另外增加了语法糖，也就是说Gradle中的Groovy是原本的Groovy的语法超集。

#### 相关命令
* 列出所有任务：
`gradle -q tasks`   -q是以quiet模式运行，只输出该task相关信息

任务有分组，不属于任何一个分组的task会显示在Other tasks中

`gradle -q tasks --all` 显示更多信息

* 任务执行
`gradle task1 task2`  一次按顺序执行多个任务，一个任务只会执行一次，例如task2依赖task1，那么task1仍然只会执行一次

**强制排除一个任务**`gradle task2 -x task1`  如果task2依赖task1，那么加上-x，则强制排除task1，只执行task2

#### 复用已存在Task
创建copyDocs任务，扩展了预置的Copy任务：
```
task copyDocs(type: Copy) {
    from 'src/main/doc'
    into 'build/target/doc'
}
```

`tasks.withType`配置某个任务，基于这个任务创建的任务都会受影响：
```
tasks.withType(Copy) {
    exclude '**/*.bak'
}

//相当于继承
task myTask(type:Copy){
    println("myTask exclude: ${myTask.excludes}")
}
```
https://docs.gradle.org/current/dsl/index.html 里面的Task Types可以看到内置的一些Task

#### 声明task的inputs和outputs
任何构建工具的一个重要部分是能够避免已经完成的工作。比如编译过程：编译完源文件后，除非发生影响输出的更改，例如修改源文件或删除输出文件，否则不需要重新编译它们。编译可能需要很长时间，因此在不需要时跳过步骤会节省大量时间。`Gradle`通过比较两个构建`task`的`inputs`和`outputs`来决定`task`是否是最新的，如果再次执行`task`，它的`outputs`对应的内容已存在并且没有改变，`inputs`对应的内容也没有发生变化，那么就可以跳过。`inputs`和`outputs`可以是变量，也可以是文件，如果是变量就根据值是否变化来确定是否跳过任务，如果是文件，那就对比文件的内容。
```
task myTask {
    def srcFile = project.file("src.txt")
    def destFile = project.file("dest.txt")

    inputs.file(srcFile)
    outputs.file(destFile)

    doLast {
        println("实际执行")
        srcFile.eachLine {line ->
            destFile.append("${line}\n")
        }
    }
}
```

### Gradle
通过Project可以获取Gradle实例，通过Gradle实例可以获取一些Gradle自身的信息，例如版本。并且可以添加监听器，用于监听各种生命周期。

### Settings
在settings.gradle中，处于Settings的作用域，可以调用Settings的方法。一些常用方法如下：
```
apply(closure)
apply(options)
findProject(path)
include(projectPaths)
includeFlat(projectNames)
project(projectDir)
```
> 官方文档：https://docs.gradle.org/current/dsl/org.gradle.api.initialization.Settings.html

### Extension
在Android项目中，我们会看到各种代码块来配置项目：
```
android {
    compileSdkVersion 30
    buildToolsVersion "30.0.3"

    defaultConfig {
        applicationId "com.xxx.myapplication"
        minSdkVersion 21
        targetSdkVersion 30
        versionCode 1
        versionName "1.0"
    }

    ...
}
```
我们也可以编写类似的DSL：
```
class Student {
    int age
    String name
}

//给project扩展一个名为student的属性，和参数为配置代码块（闭包）的方法
project.extensions.add("student", Student.class)

//实际就是project.student{...}
student {
    age 16
    name "伊泽瑞尔"
}

println(student.age)
println(student.name)
//下面的方式明确通过Extension对象也可以获取到配置信息，估计上面的简写是Gradle的语法糖
println(project.extensions.findByName("student").age)
println(project.extensions.findByName("student").name)
println(project.extensions.findByType(Student.class).name)
```
添加扩展还有其他方法：
```
project.extensions.create("student", Student.class)
project.extensions.create("student", Student.class, args)//这种重载方法，最后可以传入变长参数，会在构造Student实例的时候，选择对应的构造函数
```

#### 嵌套DSL
目前验证了三种可行的嵌套DSL定义方式。
1. 外层类的构造函数中添加内层的扩展
```
class School {
    String name
    String address
}

class Student {
    int age
    String name
    School school = new School()

    Student() {
        extensions.add("schoolConfig", School.class)
    }
}

project.extensions.add("student", Student.class)

project.student {
    age 16
    name "伊泽瑞尔"
    schoolConfig {
        name "西充中学"
        address "四川省西充县"
    }
}

println(student.age)
println(student.name)
println(student.schoolConfig.name)
println(student.schoolConfig.address)
```
2. 使用参数为`Action<Inner类型>`的函数
```
class School {
    String name
    String address

    void name(String name) {
        this.name = name
    }

    void address(String address) {
        this.address = address
    }
}

class Student {
    int age
    String name
    School school = new School()

    void schoolConfig(Action<School> action) {
        action.execute(school) // 直接执行 action 参数的 execute 方法，并传入扩展对象
    }
}

project.extensions.add("student", Student.class)

project.student {
    age 16
    name "伊泽瑞尔"
    schoolConfig {
        name "西充中学"
        address "四川省西充县"
    }
}

println(student.age)
println(student.name)
println(student.school.name)
println(student.school.address)
```
3. 使用参数为`Closure<Inner类型>`的函数
```
class School {
    String name
    String address

    void name(String name) {
        this.name = name
    }

    void address(String address) {
        this.address = address
    }
}

class Student {
    int age
    String name
    School school = new School()

    void schoolConfig(Closure<School> c) {
        org.gradle.util.ConfigureUtil.configure(c, school)
    }
}

project.extensions.add("student", Student.class)

project.student {
    age 16
    name "伊泽瑞尔"
    schoolConfig {
        name "西充中学"
        address "四川省西充县"
    }
}

println(student.age)
println(student.name)
println(student.school.name)
println(student.school.address)
```

### 命名式DSL
android的buildTypes中，我们可以写任意个构建类型，并且可以自行命名，例如debug、release、test等。要实现这种DSL，需要用到NamedDomainObjectContainer：
```
class AnimalTypes {

    //必须定义一个 name 属性
    String name

    String desc

    //构造函数必须有 name 参数
    public AnimalTypes(String name) {
        this.name = name
    }

    void desc(String desc) {
        this.desc = desc
    }

    String toString() {
        return "name = ${name}, msg = ${desc}"
    }
}

class Animal {

    //定义一个 NamedDomainObjectContainer 属性
    NamedDomainObjectContainer<AnimalTypes> animalTypes

    public Animal(Project project) {
        //通过 project.container 创建 NamedDomainObjectContainer
        NamedDomainObjectContainer<AnimalTypes> animalTypes = project.container(AnimalTypes.class)
        this.animalTypes = animalTypes
    }

    //支持DSL
    void animalTypes(Action<NamedDomainObjectContainer<AnimalTypes>> action) {
        action.execute(animalTypes)
    }
}

//创建animal扩展
def animalDsl = getExtensions().create("animal", Animal, project)

animal {
    animalTypes {
        panda {
            desc "panda is cute"
        }
        tiger {
            desc "tigers are ferocious"
        }
    }
}

task printAnimals {
    doLast {
        animalDsl.animalTypes.all { type ->
            println type
        }
    }
}
```

### Plugin
当我们想模块化或者复用构建脚本时，可以把代码写到一个单独的脚本文件中，通过本地文件或者网络URL等形式来使用它：
```
//引入脚本文件的几种形式
apply from: 'hello.gradle'
apply from: 'http://xxx.org/hello.gradle'
apply from: rootProject.file('hello.gradle')
```
在hello.gradle中，我们可以编写任意代码，并且也可以使用project实例，这个project就是引入hello.gradle的脚本中的project，相当于就是把hello.gradle的内容放到了容器脚本中。

Gradle提供了Plugin类，我们可以以Plugin类的形式提供给各个使用者复用。例如Android也提供了相应的插件，然后我们就可以在`build.gradle`中使用名为android的各种DSL来配置项目信息，并且可以使用Android插件提供的各种任务来开发、构建项目。

Plugin类的定义可以在如下三种位置：
1. 构建脚本
直接写在构建脚本中
```
class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        project.task('hello') {
            doLast {
                println 'Hello from the GreetingPlugin'
            }
        }
        //这里也可以自定义Extension等
    }
}

// Apply the plugin
apply plugin: GreetingPlugin
```

2. buildSrc project
在当前项目的根目录中，创建一个名为`buildSrc`的目录，包含`src/main/java`或者`src/main/groovy`，`kotlin`也可以，最基本的情况下，`build.gradle`可以不用，如果要添加依赖或者使用其他功能，则需要`build.gradle`。

运行 Gradle 时会检查项目中名为`buildSrc`的目录，然后 Gradle 会自动编译这部分代码，并将其放入构建脚本的类路径中，所以`buildSrc`中的代码，对整个项目里的每个构建脚本都是可见的。

所以我们可以在`buildSrc`中编写Plugin类，或者其他任意类都可以。
```
.
├── app
│   └──  ...
├── build.gradle
├── buildSrc
│   ├── build.gradle //可选
│   └── src
│       └── main
│           └── java
│               ├── Deps.java //普通类，但位于默认包名
│               └── com.hfy
│                   ├── MyTest.java //普通类
│                   └── MyPlugin.java //Gradle的Plugin子类
└── ...
```
然后在其他构建脚本中使用，例如根目录或者app目录的build.gradle：
```
import com.hfy.MyTest

task testMyBuildSrc {
    doLast {
        MyPPP.test() //需要导包
        Deps.test() //因为Deps类是默认包名，所以直接使用
    }
}

import com.hfy.MyPlugin

apply plugin: MyPlugin //引用插件类
```

除了导入类后，以类名的方式引用插件类，还可以在`buildSrc/src/`中创建`resources/META-INF/gradle-plugins/xxx.properties`文件，**properties文件名称将作为插件id使用**。
```
.
├── ...
├── buildSrc
│   ├── ...
│   └── src
│       └── main
│           ├── java
│           │   └── ...
│           └── resources
│               └── META-INF
│                   └── gradle-plugins
│                       └── com.hfy.properties
└── ...
```
com.hfy.properties文件内容如下：
```
implementation-class=com.hfy.MyPlugin
```
标识了插件id对应的插件类，所以我们现在可以通过id引用对应的插件类：
```
apply plugin: 'com.hfy'

//或者
plugins {
    id 'com.hfy'
}
```

3. 独立项目
前面两种方式，都存在只能在当前本地项目中使用的局限，如果想要将插件发布，让大家都可以使用，那么我们可以创建一个插件的项目。可以使用`gralde init`命令，并选择`Gradle plugin`项目类型，或者自己手动创建，都是可以的。我们这里命令创建项目，源码使用Java，项目如下：
```
├── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── settings.gradle
└── src
    ├── functionalTest
    │   └── ...
    ├── main
    │   ├── java
    │   │   └── hfy
    │   │       └── HfyPlugin.java
    │   └── resources
    └── test
        └── ...
```
`HfyPlugin.java`就是Plugin子类，整个项目看着和普通Java项目一样，主要是`build.gradle`内容有所不同：
```
plugins {
    // Apply the Java Gradle plugin development plugin to add support for developing Gradle plugins
    id 'java-gradle-plugin'
    //id 'groovy'  如果选择groovy语言，则还会加上groovy的插件
    //id 'org.jetbrains.kotlin.jvm' version '1.4.31' 如果选择kotlin语言，则还会加上Kotlin的插件
}

repositories {
    // Use jcenter for resolving dependencies.
    // You can declare any Maven/Ivy/file repository here.
    jcenter()
}

dependencies {
    // Use JUnit test framework for unit tests
    testImplementation 'junit:junit:4.12'
    // 如果选择Kotlin开发，那么这里还需要加上Kotlin相关的依赖，但不需要记住，可以在使用时创建一个示例工程对照就行
}

gradlePlugin {
    // Define the plugin
    plugins {
        greeting {
            id = 'hfy.greeting' //插件的id
            implementationClass = 'hfy.HfyPlugin'
        }
    }
}
```
使用`java-gradle-plugin`插件，会自动帮我们把添加gradleApi()依赖，所以要注意我们手动创建项目的话，需要加上相应的依赖。另外，`java-gradle-plugin`插件还给我们提供了gradlePlugin函数，在里面我们配置了id和implementationClass，它会帮我们创建`resources/META-INF/gradle-plugins/xxx.properties`文件，否则我们需要手动创建。

如上已经可以本地打包Jar，但一般是需要发布到本地或者远程仓库的，例如使用Maven仓库，我们还要加上maven的插件，并且使用它的发布功能：
```
plugins {
    //省略其他插件...
    id 'maven-publish'
}

//省略其他内容...
publishing {
    publications {
        // 这里的 hello 可以任意命名
        hello(MavenPublication) {
            // 插件的组id
            groupId = 'com.hfy.plugin'
            // 工件ID，或者说是插件的名字
            artifactId = 'hello'
            version = '0.0.1'
            // 组件类型，我们的插件其实就是Java组件
            from components.java
        }
    }
}
```
现在使用`./gradlew publishHelloPublicationToMavenLocal`命令可以将插件包发布到本地Maven仓库，地址是`/Users/xxx/.m2/repository/com/hfy/plugin/hello/0.0.1/hello-0.0.1.jar`。

然后在项目中使用它：
```
buildscript {
    repositories {
        mavenLocal() //因为发布到本地，所以使用本地Maven仓库
        ...
    }
    dependencies {
        ...
        classpath "com.hfy.plugin:hello:0.0.1"
    }
}

plugins {
    ...
    id 'hfy.greeting'
}
```
需要注意一下，插件的id，和插件发布到Maven配置的groupId、artifactId的区别。

如果需要发布到指定的本地仓库，或者远程仓库，可以使用如下配置，并且使用相关的publish任务即可：
```
publishing {
    ...
    repositories {
        maven {
            //如果不指定名称，默认为maven
            name = "remoteRepo"
            url = "http://xxx.xxx" //文件路径也可以
        }
        //可以配置多个仓库...
    }
}
```
> `maven-publish`插件的详细文档：https://docs.gradle.org/current/userguide/publishing_maven.html

> 发布插件的官方文档：https://docs.gradle.org/current/userguide/publishing_gradle_plugins.html

> Gradle内置的核心插件的特殊之处在于它们提供短名称，例如JavaPlugin：'java'，所有其他二进制插件必须使用插件 id 的完全限定形式（例如com.github.foo.bar）

### init Script
在普通构建脚本执行前率先执行的脚本，用得较少。[官方文档](https://docs.gradle.org/current/userguide/init_scripts.html#sec:using_an_init_script)

### Gradle 守护进程
每次初始化一个构建时，JVM都要启动一次，Gradle的依赖要载入到类加载器中，还要建立项目对象模型，这个过程要花好几秒，守护进程可以解决这个问题。守护进程以后台进程方式运行Gradle。一旦启动，gradle命令就会在后续的构建中重用之前的创建的守护进程，以避免启动时造成的开销。

在命令行中启动Gradle守护进程很简单：在运行gradle命令时加上--daemon选项。为了验证守护进程在运行，可以查看系统进程列表：
`MacOS / *nix`：ps | grep gradle
`Windows`: 任务管理器查看进程

后续的gradle命令都会重用守护进程，守护进程只会创建一次，即使在命令行中加上--daemon选型。守护进程会在3小时空闲时间只会自动过期。任何时候可以选择执行构建时不使用守护进程，只需添加命令行选型--no-daemon。要手动停止守护进程，可以执行gradle --stop。


### Property
每个`Project`和`Task`实例都提供了可以通过`getter`和`setter`方法访问的属性。一个属性可能是一个任务的描述或项目的版本。通常需要定义自己的属性，Gradle允许用户通过扩展属性自定义一些变量。
> 直接定义的一般的变量，比如def a=1，无法通过`project.a`的形式访问，
* 扩展属性
添加属性可以使用`ext`命名空间。
```
//只有在声明属性时才需要使用ext命名空间
project.ext.myProp = 'myValue'
ext {
    otherProp1 = 'aaa'
    otherProp2 = 'bbb'
}

//使用时以下方式都可以访问属性
println myProp
println project.otherProp1
println project.ext.otherProp2
```

## 依赖管理
Gradle中，使用`repositories`闭包来配置从哪里获取依赖，使用`dependencies`来配置需要依赖的类库。我们平时基本的都是使用简单的直接配置，在这种默认情况下，如果我们的项目依赖A和B，B的版本为1.0，但是A依赖了B的1.1版本，那么Gradle会最终使用版本更高的1.1版本的B。

如果需要自定义解决依赖冲突的逻辑，可以参考官方文档，了解依赖的约束控制。另外，声明版本也可以使用范围，而不是确定的版本。