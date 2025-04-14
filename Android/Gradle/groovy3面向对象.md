官方文档：https://groovy-lang.org/objectorientation.html

## 1. 类
 `Groovy`类与`Java`对应类之间的主要区别是：
* 没有可见性修饰符的类或方法会自动公开（可以使用特殊注释来实现包私有可见性）。
* 没有可见性修饰符的字段会自动转换为属性，这样可以减少冗长的代码，因为不需要显式的`getter`和`setter`方法。
* 类不需要与源文件定义具有相同的基本名称，但在大多数情况下强烈建议使用它们。
* 一个源文件可能包含一个或多个类（但如果文件包含不在类中的任何代码，则将其视为脚本）。 脚本只是具有一些特殊约定的类，并且与源文件具有相同的名称（因此不要在脚本中包含与脚本源文件同名的类定义）。

一个例子
```
class Person {                       

    String name                      
    Integer age

    def increaseAge(Integer years) { 
        this.age += years
    }
}
```

## 2.构造函数
`Groovy`支持两种构造函数调用样式：

位置参数的使用方式与使用Java构造函数的方式类似

命名参数允许您在调用构造函数时指定参数名称。

#### 2.1 位置参数
有三种形式：
1. 普通Java形式
2. as
3. 强制
```
class PersonConstructor {
    String name
    Integer age

    PersonConstructor(name, age) {          
        this.name = name
        this.age = age
    }
}

def person1 = new PersonConstructor('Marie', 1)   //1
def person2 = ['Marie', 2] as PersonConstructor   //2
PersonConstructor person3 = ['Marie', 3]  //3
```

#### 2.3 命名参数
如果没有声明（或只有无参数）构造函数，则可以通过以map（属性/值对）的形式传递参数来创建对象。
```
class PersonWOConstructor {                                  
    String name
    Integer age
}

def person4 = new PersonWOConstructor()                      
def person5 = new PersonWOConstructor(name: 'Marie')         
def person6 = new PersonWOConstructor(age: 1)                
def person7 = new PersonWOConstructor(name: 'Marie', age: 2)
```
然而，重要的是要强调，这种方法为构造函数调用者提供了更多的功能，同时增加了调用者的责任，以使名称和值类型正确。 因此，如果需要更大的控制，则可能优选使用位置参数来声明构造函数。

## 3.方法
#### 3.1 方法定义
&emsp;&emsp;使用返回类型或使用def关键字定义方法，以使返回类型无类型化。 方法还可以接收任意数量的参数，这些参数可能没有显式声明其类型。 Java修饰符可以正常使用，如果没有提供可见性修饰符，则该方法是公共的。

&emsp;&emsp;`Groovy`中的方法总是返回一些值。 如果未提供`return`语句，则将返回在执行的最后一行中计算的值。 

#### 3.2 命名参数
与构造函数一样，也可以使用命名参数调用常规方法。 为了支持这种表示法，使用了一种约定，其中方法的第一个参数是Map。 在方法体中，可以像在法线贴图中一样访问参数值（map.key）。 如果该方法只有一个Map参数，则必须命名所有提供的参数。
```
def foo(Map args) { "${args.name}: ${args.age}" }
foo(name: 'Marie', age: 1)
```
`混合命名和位置参数`
命名参数可以与位置参数混合。 在这种情况下，将`Map`参数作为第一个参数。 调用方法时提供的位置参数必须按顺序排列。 命名参数可以在任何位置， 它们被分组到Map中并自动作为第一个参数提供。
```
def foo(Map args, Integer number) { "${args.name}: ${args.age}, and the number is ${number}" }
foo(name: 'Marie', age: 1, 23)  
foo(23, name: 'Marie', age: 1)
```
如果`Map`不是第一个参数，那么必须传入一个`Map`类型，而不是命名参数：
```
def foo(Integer number, Map args) { "${args.name}: ${args.age}, and the number is ${number}" }
foo(23, [name: 'Marie', age: 1])
```

#### 3.3 域和属性
###### 3.3.1 域（字段）
域是类的成员，有以下特征：
* 必须有访问修饰符（`public`, `protected`,or `private`）
* 一个或多个可选修饰符（`static`, `final`,` synchronized`）
* 一个可选的类型
* 一个必须的名称

###### 3.3.2 属性
属性是类的外部可见特性，Java是私有字段配合getter/setter的JavaBean规范， Groovy遵循这些相同的约定，但提供了一种更简单的方法来定义属性：
* 无访问修饰符（没有`public`, `protected`,or `private`）
* 一个或多个可选修饰符（`static`, `final`,` synchronized`）
* 一个可选的类型
* 一个必须的名称

`Groovy`会适当地生成`getter/setter`，如下会隐藏生成`private String name`实例域（字段）和`getName`和`setName`方法，`age`也是。
```
class Person {
    String name                             
    int age                                 
}
```
如果属性声明`final`则不会生成`set`方法

如下访问会隐式调用`getter/setter`
```
p.name = 'Marge'                        
assert p.name == 'Marge'    
```