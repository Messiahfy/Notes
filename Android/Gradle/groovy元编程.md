Groovy支持运行时和编译时元编程

https://groovy-lang.org/metaprogramming.html
## 运行时元编程

Groovy作为动态语言，使用更灵活，可以通过metaClass、setProperty等方式反射、代理。Groovy 编译器在编译 Groovy 代码的时候，会做额外的包装处理。

* POJO - 常规 Java 对象，其类可以用 Java 或 JVM 的任何其他语言编写。
* POGO - 一个 Groovy 对象，其类是用 Groovy 编写的。默认情况下，它扩展java.lang.Object并实现了groovy.lang.GroovyObject接口。
* Groovy Interceptor - 实现groovy.lang.GroovyInterceptable接口并具有方法拦截功能的Groovy 对象，这将在GroovyInterceptable部分中讨论。

对于POJO 使用 groovy.lang.MetaClassRegistry 得到 MetaClass  GroovyObjectSupport
```
    public static MetaClass getMetaClass(Object object) {
        return object instanceof GroovyObject ? ((GroovyObject)object).getMetaClass() : ((MetaClassRegistryImpl)GroovySystem.getMetaClassRegistry()).getMetaClass(object);
    }
```

MetaClassImpl默认实现，还可以自己实现。可以继承MetaClassImpl, DelegatingMetaClass, ExpandoMetaClass, or ProxyMetaClass，否则需要自己实现更多的逻辑


在Groovy语言中，每个对象都有一个名称为metaClass的MetaClass类的对象，

```
public interface MetaClass extends MetaObjectProtocol {
    Object invokeMethod(Class var1, Object var2, String var3, Object[] var4, boolean var5, boolean var6);

    Object getProperty(Class var1, Object var2, String var3, boolean var4, boolean var5);

    void setProperty(Class var1, Object var2, String var3, Object var4, boolean var5, boolean var6);

    Object invokeMissingMethod(Object var1, String var2, Object[] var3);

    Object invokeMissingProperty(Object var1, String var2, Object var3, boolean var4);

    Object getAttribute(Class var1, Object var2, String var3, boolean var4);

    void setAttribute(Class var1, Object var2, String var3, Object var4, boolean var5, boolean var6);

    void initialize();

    List<MetaProperty> getProperties();

    List<MetaMethod> getMethods();

    ClassNode getClassNode();

    List<MetaMethod> getMetaMethods();

    int selectConstructorAndTransformArguments(int var1, Object[] var2);

    MetaMethod pickMethod(String var1, Class[] var2);
}
```


```
public interface GroovyObject {
    @Internal
    default Object invokeMethod(String name, Object args) {
        return this.getMetaClass().invokeMethod(this, name, args);
    }

    @Internal
    default Object getProperty(String propertyName) {
        return this.getMetaClass().getProperty(this, propertyName);
    }

    @Internal
    default void setProperty(String propertyName, Object newValue) {
        this.getMetaClass().setProperty(this, propertyName, newValue);
    }

    MetaClass getMetaClass();

    void setMetaClass(MetaClass var1);
}
```

```
public interface GroovyInterceptable extends GroovyObject {
}
```
用于标记继承自GroovyObject，并且用于通知Groovy runtime 所有方法应该通过Groovy runtime 的方法dispatcher机制拦截


`MetaClassImpl`包含以下方法，都会返回CallSite：
```
createPojoCallSite
createStaticSite
createPogoCallSite
createPogoCallCurrentSite
createConstructorSite
```  
CallSite有很多实现类，比如PogoMetaClassSite。可以委托方法调用，实现动态特性。

Groovy 编译器编译后的代码，所有方法调用都会通过 Groovy 的元编程相关类包装，从而实现动态特性。

参考文章：https://juejin.cn/post/6864734617069928456

### Categories
类似Kotlin扩展函数。比如一个类不是我们编写的，需要添加扩展方法。

参考官方文档即可：https://groovy-lang.org/metaprogramming.html#categories

系统中包括几个Categories，用于向类添加功能，以使这些类在Groovy环境中更易于使用：

groovy.time.TimeCategory


ExpandoMetaClass允许通过使用一个整洁的闭包语法动态添加或更改方法，构造函数，属性，甚至静态方法。

扩展模块，类似扩展方法

For Groovy to be able to load your extension methods, you must declare your extension helper classes. You must create a file named org.codehaus.groovy.runtime.ExtensionModule into the META-INF/groovy directory:
```
moduleName=Test module for specifications
moduleVersion=1.0-test
extensionClasses=support.MaxRetriesExtension
staticExtensionClasses=support.StaticStringExtension
```


## 编译时元编程
编译时生成代码。编译时添加的方法在Java中也能使用


### 生成代码
自带的注解
@groovy.transform.ToString
```
import groovy.transform.ToString

@ToString
class Person {
    String firstName
    String lastName
}

def p = new Person(firstName: 'Jack', lastName: 'Nicholson')
assert p.toString() == 'Person(Jack, Nicholson)'
```
将生成可读的toString方法

@groovy.transform.EqualsAndHashCode
```
import groovy.transform.EqualsAndHashCode

@EqualsAndHashCode
class Person {
    String firstName
    String lastName
}

def p1 = new Person(firstName: 'Jack', lastName: 'Nicholson')
def p2 = new Person(firstName: 'Jack', lastName: 'Nicholson')

assert p1==p2
assert p1.hashCode() == p2.hashCode()
```
生成equals and hashCode 



@groovy.transform.TupleConstructor 生成构造函数
```
```

@groovy.transform.MapConstructor

@groovy.transform.Canonical

@groovy.transform.InheritConstructors 生成继承父类的全部构造函数

@groovy.lang.Category 生成 Groovy categories

@groovy.transform.IndexedProperty

@groovy.lang.Lazy 懒加载

@groovy.lang.Newify

@groovy.transform.Sortable

@groovy.transform.builder.Builder

@groovy.transform.AutoImplement

@groovy.transform.NullCheck


@groovy.transform.BaseScript 设置让类继承自定义的script基类，而不是groovy.lang.Script

@groovy.lang.Delegate 
```
class Event {
    @Delegate Date when
    String title
}

class Event {
    Date when
    String title
    boolean before(Date other) {
        when.before(other)
    }
    // ...
}
```

@groovy.transform.Immutable 简化不可变类的创建，‘

@groovy.lang.Singleton

......等等


### 开发
