https://www.bilibili.com/video/BV1DE411Z7nt?p=2

https://docs.gradle.org/current/userguide/getting_started.html

## 2.task
```
task  helloWorld {
    doLast {    //doLast是task的action（动作）
          println 'Hello world'
    }
}
```
执行该task：`gradle helloWorld`

可以使用<<代替doLast
```
task  helloWorld << {
      println 'Hello world'
}
```
`gradle taskname` 执行某任务，并且会先执行其依赖的任务

任务依赖
```
task0.dependsOn startSession
task2.dependsOn task1, task0
```

* 列出所有任务：
`gradle -q tasks`   -q是以quiet模式运行，只输出该task相关信息

任务有分组，不属于任何一个分组的task会显示在Other tasks中

`gradle -q tasks --all` 显示更多信息

* 任务执行
`gradle task1 task2`  一次按顺序执行多个任务，一个任务只会执行一次，例如task2依赖task1，那么task1仍然只会执行一次

**强制排除一个任务**`gradle task2 -x task1`  如果task2依赖task1，那么加上-x，则强制排除task1，只执行task2

* 命令行选项

* Gradle 守护进程
&emsp;&emsp;每次初始化一个构建时，JVM都要启动一次，Gradle的依赖要载入到类加载器中，还要建立项目对象模型，这个过程要花好几秒，守护进程可以解决这个问题。守护进程以后台进程方式运行Gradle。一旦启动，gradle命令就会在后续的构建中重用之前的创建的守护进程，以避免启动时造成的开销。
&emsp;&emsp;在命令行中启动Gradle守护进程很简单：在运行gradle命令时加上--daemon选项。为了验证守护进程在运行，可以查看系统进程列表：
`MacOS / *nix`：ps | grep gradle
`Windows`: 任务管理器查看进程

后续的gradle命令都会重用守护进程，守护进程只会创建一次，即使在命令行中加上--daemon选型。守护进程会在3小时空闲时间只会自动过期。任何时候可以选择执行构建时不使用守护进程，只需添加命令行选型--no-daemon。要手动停止守护进程，可以执行gradle --stop。


## 3.一个Java项目范例
创建`build.gradle`文件，文件中只需一行`apply plugin: 'Java'`。默认情况下，插件会到src/main/Java目录下查找源代码。此时目录如下：
|--build.gradle
|--src
&emsp;&emsp;|--main
&emsp;&emsp;&emsp;&emsp;|--java
省略java目录中的com.xx.xx

#### 3.1 构建项目
Java插件提供了一个任务叫`build`。这个`build`任务会以正确的顺序编译源代码、运行测试、组装JAR文件。执行`build`任务会执行一系列任务（其依赖的任务）。执行任务输出的log中，某些任务被标记为UP-TO-DATE消息，表示这个任务被跳过了。Gradle支持自动鉴别不需要被运行的任务

#### 3.2 定制项目
Java插件是一个很灵活的框架，它会给项目的很多方面设置有意义的默认值，比如项目结构。如何知道什么是可配置的？Gradle构建语言指导http://www.gradle.org/docs/current/dsl/，运行gradle properties会显示一个可配置标准和插件属性的列表，同时会显示它们的默认值。可以通过扩展初始构建脚本来定制你的项目。
* 修改项目和插件属性
为了能够从`JAR`文件启动应用，清单文件`MANIFEST.MF`需要包含信息头`Main-Class`。下面展示在构建脚本中如何配置。
```
version = 0.1  //定义项目版本
sourceCompatibility = 1.6  //设置Java版本编译兼容1.6
jar {
    manifest {
        //将Main-Class头添加到JAR文件代码清单中
        attribute `Main-Class`: `com.manning.gia.todo.ToDoApp`
    }
}
```
在组装完JAR文件后，会看到版本号加到了JAR文件的名字中，现在名字是todo-app-0.1.jar，而不是todo-app.jar。现在JAR中包含了主类头属性，可以通过java -jar build/libs/todo-app-0.1.jar命令运行应用。

* 改造遗留项目
&emsp;&emsp;假设需要把源代码放置在src目录下，而不是src/main/java。同样的道理，也适用于改变默认的测试代码目录。另外，如果让Gradle将输出结构放置到out目录，而不是build。
```
sourceSets {
    main {
        java {
            srcDirs = ['src']    //用不同目录的列表代替默认的源代码目录
        }
    }
    test {
        java {
            srcDirs = ['test']    //用不同目录的列表代替约定的测试代码目录
        }
    }
}
buildDir = 'out'    //改变项目输出属性（目录）到out目录
```
定制一个构建的关键就是了解潜在的属性和DSL元素。

## 3.3 配置和使用外部依赖
&emsp;&emsp;如何告诉Gradle去哪引用第三方库，两个DSL配置元素：`repositories`和`dependencies`。
* 定义仓库
&emsp;&emsp;在Java中，依赖都是以JAR文件的形式发布和使用，许多类库都可以在仓库中找到，仓库可以是一个文件系统或者一个中心服务器。Gradle要求定义至少一个仓库来使用依赖。为此，可以使用Maven Central：
```
repositories {
    //配置对Mavan Central 2仓库http://repol.maven.org/maven2访问的快捷方式
    mavenCentral()
}
```
定义好仓库后，就可以声明依赖类库了
* 定义依赖
```
dependencies {
    implementation group: 'com.android.support', name: 'appcompat-v7', version: '28.0.0'
    //或简写
    implementation 'com.android.support:appcompat-v7:28.0.0'
}
```
除了`implementation`，还有`compile`、`api`等

* 解析依赖
&emsp;&emsp;Gradle会自动检测到一个新的依赖添加到项目中。如果依赖没有被成功解析，那么就会在下一个需要使用该依赖的任务启动时去下载它。

#### 3.4 Gradle包装器
&emsp;&emsp;`Gradle Wrapper`能够让机器在没有安装Gradle运行时的情况下运行Gradle构建。它也让构建脚本运行在一个指定的Gradle版本上。它是通过自动从中心仓库下载Gradle运行时，解压和使用来实现的。最终的目标是创造一个独立于系统、系统配置和Gradle版本的可靠和可重复的构建。

###### 3.4.1 配置包装器
&emsp;&emsp;在项目中配置包装器，只需做两件事情：创建一个包装器任务和执行任务生成包装器文件。
1. 在`build.gradle`中添加如下任务：
```
task wrapper(type: Wrapper){
    gradleVersion = '4.10.1'
}
```
定义一个类型为Wrapper的任务，通过gradleVersion属性指定想要的Gradle版本。不要求该任务的名字为wrapper，任何名字都可以，但wrapper是默认约定。

2. 在shell中执行命令：`gradle wrapper`
执行任务后，会发现包装器文件在build.gradle的同一级目录中。
```
|--build.gradle
|--gradle
    |--wrapper
        |--gradle-wrapper.jar  //微类库包含下载和解包Gradle运行时的逻辑
        |--gradle-wrapper.properties  //元信息包含已下载Gradle运行时的存储位置和原始URL
|--gradlew          //执行Gradle命令的包装器脚本
|--gradlew.bat
```
&emsp;&emsp;只需要在项目中运行一次gradle wrapper命令。从那时开始，就可以使用包装器的脚本执行构建了。也可以使用build setup插件，而不需要手动创建一个包装器任务和执行它来下载相关文件。

###### 3.4.2 定制包装器
&emsp;&emsp;例如为政府机构工作时，访问外网是禁止的，此时可以改变默认配置，将它指向一个存有运行时发布文件的企业内部服务器。同时还可以改变本地的存储路径：
```
task(type: Wrapper) {
    gradleVersion = '4.10.1'  //请求的Gradle版本
    distributionUrl = 'http://my.com/gradle/dists'//获取Gradle包装器的URL
    distributionPath = 'gradle-dists' //包装器被解压缩后存放的相对路径
}
```

## 4.构建脚本概要
#### 4.1 构建块
&emsp;&emsp;每个Gradle构建都包含三个基本构建块：`project`、`task`和`property`。每个构建包含至少一个`project`（多项目构建中一个`project`可以依赖于其他的`project`），进而又包含一个或多个`task`。`project`和`task`暴露的属性可以用来控制构建。
&emsp;&emsp;Gradle API中有相应的类来表示`project`和`task`。

###### 4.1.1 Project
在`Gradle`术语中，一个`Project`（项目）代表一个正在构建的组件（比如一个JAR文件），或一个想要完成的目标，如部署应用程序。每个`Gradle`构建脚本至少定义一个项目。当构建进程启动后，`Gradle`基于`build.gradle`中的配置实例化`org.gradle.api.Project`类，并且能够通过`project`变量使其隐式可用。下面列出重要的方法：
```
<<interface>> Project
----------------------
//配置构建脚本
apply(Map<String,?> options) //继承自PluginAware，还有其他重载
buildScript(Closure configureClosure) 

//依赖管理
dependencies(Closure configureClosure)
configurations(Closure configureClosure) 
getDependencies()
getConfigurations()

//getter/setter属性
getAnt()
getName()
getDescription()
getGroup()
getPath()
getVersion()
getLogger()
setDescription(String description)
setVersion(Object version)

//创建文件
file(Object path)
file(Object... paths)
fileTree(Object baseDir)

//创建任务
task(String name)
task(String name, Closure c)
task(Map<String, ?> args, String name)
task(Map<String, ?> args, String name, Closure c)
```
&emsp;&emsp;一个`project`可以创建新的`task`，添加依赖关系和配置，并应用插件和其他的构建脚本。它的许多属性，如name和description可以通过getter和setter方法访问。
&emsp;&emsp;在构建中，`Project`实例可以让我们通过代码来访问Gradle的所有特性，比如`task`的创建和依赖管理。当访问属性和方法时，不需要使用`project`变量----它会假设你是指向`Project`实例。
```
setDescription("myProject")  //在不显式使用project变量时调方法
//不使用project变量的情况下通过Groovy语法访问name和description属性
println "Description of project $name：" + project.description
```
> 上面两行代码，调用的都是project的方法，所以如果在某闭包中调用，groovy的语法作用域还不清楚***************待修改

&emsp;&emsp;前面的章节都是单项目构建，`Gradle`也支持多项目构建。软件系统越复杂，就越想要把它分解为多个模块化的功能，这些模块可以相互依赖。每个分解的部分将被表示为一个`Gradle`项目，并且有自己独立的`build.gradle`脚本。第6章详细介绍。

###### 4.1.2 Task
任务的重要功能：任务动作（`Action`）和任务依赖（`dependency`）。任务动作定义了一个当任务执行时的最小工作单元。可以简单到打印“Hello World！”，或复杂到编译Java源代码。运行`task`前可能需要运行另一个`task`，尤其是需要前一个的输出作为后一个的输入时。比如打包`JAR`文件前需要先编译Java源代码。`org.gradle.api.Task`接口部分API如下：
```
<<interface>> Task
-------------------------
dependsOn(Object... paths)  //task依赖

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
###### 4.1.3 Property
&emsp;&emsp;每个`Project`和`Task`实例都提供了可以通过`getter`和`setter`方法访问的属性。一个属性可能是一个任务的描述或项目的版本。通常需要定义自己的属性，Gradle允许用户通过扩展属性自定义一些变量。
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

* Gradle属性
属性也可以通过属性文件来提供，Gradle的属性可以通过在`gradle.properties`文件中声明直接添加到项目中，这个文件位于`~/.gradle`或者`项目的根目录`下。
例如在`gradle.properties`文件中声明如下：
```
a = 1
b = 2
```
可以用两个形式访问
```
println project.a
println "$b"
```

* 声明属性的其他方式
&emsp;&emsp;对于前面两种方式，大多用来声明自定义变量及其值。Gradle也提供了很多其他方式为构建提供属性，例如：
项目属性通过-P 命令行提供
系统属性通过-D 命令行提供

#### 4.2 使用Task
&emsp;&emsp;默认情况下，每个新建的`task`都是`org.gradle.api.DefaultTask`类型，标准的`org.gradle.api.Task`实现，`DefaultTask`中的所有属性都是`private`，意味着只能通过`getter/setter`来访问，但是`Groovy`提供了语法糖，可以直接通过属性名来使用属性。在底层，`Groovy`为替我们调用这些方法。

###### 4.2.1 声明task动作
动作（`action`）就是在`task`中合适的地方放置构建逻辑。`Task`接口提供了两个相关的方法来声明`task`动作：`doFirst(Closure)`和`doLast(Closure)`。当`task`被执行的时候，动作逻辑被定义为闭包参数被依次执行。
```
version = `0.1-SNAPSHOT`

task printVersion {
    doLast {
        println "Version: $version"
    }
}
```
#### 理解task配置
开始编写代码前，先创建一个名为`version.properties`的属性文件，并且设置版本属性，如下：
```
major = 0
minor = 1
release = false
```
然后声明一个task，从属性文件中读取版本类别，并赋值给项目的版本属性。此时task中没有定义动作，不是动作的代码称为`task配置`
```
ext.versionFile = file(`version.properties`)

task loadVersion {
    project.version = readVersion()    //task配置
}
//ProjectVersion是一个自定义的POJO类（省略），
//将它赋值给version，Gradle总是会调用它的 toString 方法获取version值
ProjectVersion readVersion() {  
//省略读取文件代码
}
```


Gradle构建生命周期阶段

无论什么时候执行Gradle构建，都会运行三个不同的生命周期阶段：初始化、配置和运行。

* `初始化阶段`，Gradle为项目创建了一个`Project`实例。在给定的构建脚本中只定义了一个项目。在多项目构建中，这个构建阶段变得更加重要。根据你正在执行的项目，Gradle 找到哪些项目依赖需要参与到构建中。**注意，在这个构建阶段当前已有的构建脚本都不会被执行。**

* `配置阶段`，Gradle构造了一个模型来表示任务，并参与到构建中来。增量式构建特性决定了模型中的`task`是否需要被运行。这个阶段非常适合于为项目或指定`task`设置所需的配置。
> 注意：项目的每一次构建的任何配置代码都可以被执行----即使你只执行 gardle tasks。

*   `执行阶段`，所有的`task`都应该以正确的顺序被执行。执行顺序是由他们的依赖决定的。如果任务被认为没有被修改过，将被跳过。

#### 声明task的inputs和outputs
&emsp;&emsp;任何构建工具的一个重要部分是能够避免已经完成的工作。比如编译过程：编译完源文件后，除非发生影响输出的更改，例如修改源文件或删除输出文件，否则不需要重新编译它们。编译可能需要很长时间，因此在不需要时跳过步骤会节省大量时间。
&emsp;&emsp;`Gradle`通过比较两个构建`task`的`inputs`和`outputs`来决定`task`是否是最新的，如果最后一次执行`task`后，它的`inputs`和`outputs`都没有发生变化，那么就可以跳过。

输入可以是一个目录、一个或多个文件，或者是一个任意属性。输出是一个目录或者 一个或多个文件。`inputs`和`outputs`是`DefaultTask`的两个属性，对应`TaskInputs`和`TaskOutputs`接口。
```
task makeReleaseVersion (group: `versioning`, description: `Make project a release version.`) {
    //在配置阶段声明inputs和outputs
    inputs.property(`release`, version.release)
    outputs.file versionFile

    doLast {
        version.release = true
        ant.propertyfile(file: versionFile) {
            entry(key: `release`, type: `string`, operation: `=`, value: `true`)
        }
    }
}
```
现在，执行task两次，就会跳过第二次执行，因为第二次执行的时候，inputs和outputs指向的内容并没有变。

#### 编写和自定义task
&emsp;&emsp;Gradle为构建脚本中每个简单的`task`都创建了一个`DefaultTask`类的实例。当创建一个自定义的任务时，需要做的是，创建一个继承`DefaultTask`的类。
```
class ReleaseVersionTask etends DefaultTask {
    @Input Boolean release
    @OutputFile File destFile        //通过注解声明 task 的输入输出

    ReleaseVersionTask(){
        group = `versioning`
        //在构造器中设置 task 的 group 和 description 属性
        description = `Makes project a release version.`
    }
    
    @TaskAction    //使用注解声明将被执行的方法
    void start() {
        project.version.release = true
        ant.propertyfile(file: destFile) {
            entry(key: `release`, type: `string`, operation: `=`, value: `true`)
        }
    }
}
```
在上面代码中，没有使用`DefaultTask`类的属性来声明它的输入输出，而是使用`org.gradle.api.tasks`包下的注解。为属性添加输入输出注解不是唯一的选择，也可以选择为`getter`方法添加注解。

###### 使用自定义task
&emsp;&emsp;上面通过创建一个动作方法和暴露它的配置属性，我们实现了一个自定义的`task`类。要如何使用它呢？在构建脚本中，需要创建一个`ReleaseVersionTask `类型的`task`,并且通过为它的属性赋值来设置输入和输出，如下所示，认为这是创建一个特定类的新实例，并且在构造器中为它的属性设置值。
```
//定义一个增强的ReleaseVersionTask类型的task
task makeReleaseVersion(type: ReleaseVersionTask){
    release = version.release  //设置自定义 task 的属性
    destFile = versionFile
}
```
与简单的task相比，增强的task的一个巨大优势在于暴露的属性可以被单独赋值。比如命名版本文件改变了，此时可以使`destFile = file(`xxx.properties`)`。

#### Gradle的内置task类型
&emsp;&emsp;Gradle封装了大量开箱即用的自定义类作为常用功能，比如复制和删除文件，或者创建一个ZIP归档。

首先展示用于发布项目的task依赖关系例子，后面的依赖前面的：
1. makeReleaseVersion
2. war
3. createDistribution
4. backupReleaseVersion
5. release

&emsp;&emsp;Gradle的内置`task`类型都是`DefaultTask`的派生类。因此，它们可以被构建脚本中的增强的task使用。下面代码显示了在产品发布过程中用到的`task`类型`Zip`和`Copy`:使用task类型来备份ZIP发布包
```
task createDistribution(type: Zip, dependOn: makeReleaseVersion) {
    from war.outputs.files    //隐式引用 War task 的输出
    from(sourceSet*.allSouce){
        into `src`    //把所有源文件都放到ZIP文件的src目录下
    }

    from(rootProject){
        include versionFile.name    //为ZIP文件添加版本文件
    }
}

task backupReleaseDistribution(type: Copy){
    from createDistribution.outputs.files    //隐式引用createDistribution的输出
    into "$buildDir/backup"
}

task release (dependOn: backupReleaseDistribution) << {
    logger.quite `Releasing the project...`
}
```
在这个例子中，使用了不同的方式来告诉`Zip`和`Copy`task包括哪些文件以及它们被放到了哪里。这里使用的许多方法都继承自父类`AbstractCopyTask`。了解更多需查阅Javadocs文档


###### task依赖推断
上面代码的两个task之间的依赖关系是通过`dependsOn`方法显示声明的，然而，有些task并不直接依赖其他task（比如`createDistribution `对于`war`），其实通过使用一个task的输出作为另一个task 的输入，Gradle就可以推断出依赖关系，因此，所依赖的task会自动运行。

#### task规则
可用于将多个类似的task写成一个task，暂不深入。

#### 在buildSrc目录下构建代码
&emsp;&emsp;可以将构建代码放到`buildSrc`目录中，groovy放到`src/main/groovy`，位于这些目录下的代码都会被自动编译，并且被加入到Gradle构建脚本的classpath中。
![目录结构](https://upload-images.jianshu.io/upload_images/3468445-b0bde1b4a3103630.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在构建脚本中定义一个类和一个独立的源文件的区别在于独立源文件需要从Gradle API导入类。相应的，构建脚本需要从buildSrc导入编译过的类。

#### 挂接到构建声明周期
&emsp;&emsp;作为一个构建脚本开发人员，不能局限于编写在不同的构建阶段执行的task动作或者配置逻辑。有时候当一个特定的生命周期事件发生时你可能想要执行代码。一个生命周期可能发生在某个构建阶段之前、期间或者之后。在执行阶段之后发生的生命周期事件是构建的完成。
&emsp;&emsp;例如想在构建失败后给团队发送邮件。有两种方式可以编写回调生命周期事件：在闭包中，或者通过Gradle API所提供的监听器接口实现。
![构建生命周期钩子示例](https://upload-images.jianshu.io/upload_images/3468445-2327ba65f6d80e88.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
许多生命周期回调方法被定义在`Project`和`Gradle`接口中。

###### 除了hook，也可以使用生命周期监听器



