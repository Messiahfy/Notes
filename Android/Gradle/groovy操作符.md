## 1.对象操作符
#### 1.1 安全导航操作符（Safe Navigation operator）
被用于避免`NullPointerException`，
```
def name = person?.name              //假设person为null        
assert name == null
```
返回`null`结果，而不是抛出错误，使用`null-safe`运算符可防止出现`NullPointerException`。

#### 1.2 直接公有域访问操作符
使用`user.name`默认调用`getter`，使用`@`操作符则强制使用域本身而不是`getter`（有时`getter`返回的值经过自定义，可能和域本身有差别）
```
assert user.@name == 'Bob' 
```

#### 1.3 方法指针操作符
方法指针运算符（`.＆`）调用用于存储对变量中方法的引用，以便稍后调用它：
```
def str = 'example of method reference'            
def fun = str.&toUpperCase                         
def upper = fun()                                  
assert upper == str.toUpperCase()
```

使用方法指针有许多优点。 首先，这种方法指针的类型是groovy.lang.Closure，因此它可以在任何地方使用闭包。 特别是，它适合转换现有方法以满足策略模式的需要：
```
def transform(List elements, Closure action) {                    
    def result = []
    elements.each {
        result << action(it)
    }
    result
}
String describe(Person p) {                                       
    "$p.name is $p.age"
}
def action = this.&describe                                       
def list = [
    new Person(name: 'Bob',   age: 42),
    new Person(name: 'Julia', age: 35)]                           
assert transform(list, action) == ['Bob is 42', 'Julia is 35']
```

## 2.其他操作符
#### 2.1 Spread operator
用于调用聚合对象的所有项目上的操作。 它相当于对每个项目调用操作并将结果收集到列表中：
```
class Car {
    String make
    String model
}
def cars = [
       new Car(make: 'Peugeot', model: '508'),
       new Car(make: 'Renault', model: 'Clio')]       
def makes = cars*.make                                
assert makes == ['Peugeot', 'Renault']
```

#### 2.2 Call operator
调用操作符`()`用于隐式调用名为`call`的方法。 对于定义调用方法的任何对象，可以省略.call部分并使用调用运算符：
```
class MyCallable {
    int call(int x) {           
        2*x
    }
}

def mc = new MyCallable()
assert mc.call(2) =
```