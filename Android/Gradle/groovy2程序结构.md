## 1.脚本与类
#### 1.1 public static void main vs script
groovy同时支持脚本和类的写法

Main.groovy
```
class Main {                                    
    static void main(String... args) {          
        println 'Groovy world!'                 
    }
}
```
或者
Main.groovy
```
println 'Groovy world!'
```
脚本可以被视为一个类而不需要声明它，但有一些差异。

#### 1.2 Script class
一个脚本总是会被编译为一个类（`groovy.lang.Script`：代表`Groovy`  脚本的对象）， Groovy编译器将为您编译该类，并将脚本的主体复制到`run`方法中。 因此，前面的示例被编译为如下所示：

Main.groovy
```
import org.codehaus.groovy.runtime.InvokerHelper
class Main extends Script {                     
    def run() {                                 
        println 'Groovy world!'                 
    }
    static void main(String[] args) {           
        InvokerHelper.runScript(Main, args)     
    }
}
```
如果脚本位于文件中，则使用该文件的基本名称来确定生成的脚本类的名称。 在此示例中，如果文件的名称为Main.groovy，则脚本类将为Main。

##### 1.3 方法
生成的脚本类将所有方法都包含在脚本类中，并将所有脚本体组装到run方法中：
```
println 'Hello'                                 

int power(int n) { 2**n }                       

println "2^6==${power(6)}"
```
此代码在内部转换为：
```
import org.codehaus.groovy.runtime.InvokerHelper
class Main extends Script {
    int power(int n) { 2** n}                   
    def run() {
        println 'Hello'                         
        println "2^6==${power(6)}"              
    }
    static void main(String[] args) {
        InvokerHelper.runScript(Main, args)
    }
}
```
即使Groovy从您的脚本创建了一个类，它对用户来说也是完全透明的。 特别是，脚本被编译为字节码，并保留行号。 这意味着如果在脚本中抛出异常，堆栈跟踪将显示与原始脚本对应的行号，而不是我们显示的生成代码。

#### 1.4 变量
```
int x = 1
int y = 2
assert x+y == 3
```

```
x = 1
y = 2
assert x+y == 3
```
这两种类型定义都可以，但是存在一些差异。

* 第一种情况，是局部变量， 它将在编译器生成的`run`方法中声明，并且在脚本主体外部不可见。 特别是，这样的变量在脚本的其他方法中是不可见的。
```
def x

String line() {
    "="*x
}
```
上面将生成下面的代码，变量x在line()方法中不可见（但可以用作参数传递）
```
class MyScript extends Script {

    String line() {
        "="*x
    }

    public def run() {
        def x 
    }
}
```
* 第二种情况暂不清楚

如果想让变量成为类的实例域，可以使用`@Field`注解
```
@Field def x

String line() {
    "="*x
}
```
将生成下面代码，此时可以在line()中访问x
```
class MyScript extends Script {

    def x

    String line() {
        "="*x
    }

    public def run() {
    }
}
```