## 1. 概要
Spring使创建Java企业应用程序变得容易。Spring框架分为多个模块，应用程序可以选择所需的模块。Core模块是最关键的部分，包括依赖项注入（控制反转）和面向切面编程（AOP）等机制。除此之外，Spring框架的其他模块还提供了事务性数据和持久化，以及基于Servlet的Spring MVC Web框架等。

引入依赖，这里直接使用SpringMVC的仓库坐标，因为它依赖Core等模块，所以会将Core等模块都引入：
```
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-webmvc</artifactId>
  <version>5.3.0</version>
</dependency>
```

## 2. 控制反转（IoC）容器
控制反转是一种设计思想，依赖注入是实现控制反转的一种方式。使用Spring，可以把对象的创建、管理交给Spring。

ApplicationContext接口代表IoC容器，容器通过读取配置元数据来决定如何实例化、配置和组装对象，配置元数据可以通过XML、Java注解或者Java代码表示。配置元数据可以表达提供的对象和对象之间的依赖关系。ApplicationContext接口有多种实现，常用的是`ClassPathXmlApplicationContext`或者`FileSystemXmlApplicationContext`。

### 2.1 XML配置元数据示例
首先定义一个User类：
```
public class User {
    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

然后在maven项目的resources里面创建beans.xml文件：
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="user" class="com.hfy.User">
        <property name="name" value="颖颖颖"/>
    </bean>
</beans>
```

最后使用ApplicationContext获取实例：
```
ApplicationContext applicationContext = new ClassPathXmlApplicationContext("beans.xml");
User user = (User) applicationContext.getBean("user");
```

### 2.2 xml配置说明
#### 别名
在xml中使用alias标签，则通过别名也能获取到实例
```
<alias name="user" alias="user2"/>
```

#### bean的配置
```
<bean id="user" class="com.hfy.User">
    <property name="name" value="颖颖颖"/>
</bean>
```
* id：bean的标识符，通过它获取实例
* class：bean对象对应的全限定类名
* name：也是别名，并且可以设置多个，用逗号隔开

#### import
如果项目有多个spring的xml配置文件，可以使用import将多个xml导入到一个总的xml文件中，此时创建ApplicationContext时传入总的xml路径即可
```
// applicationContext.xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <import resource="beans1.xml"/>
    <import resource="beans2.xml"/>
</beans>
```

## 3. 依赖注入方式
### 构造函数注入
前面的例子中，展示的基于setter的方式注入依赖，这里展示基于构造函数的方式注入依赖。如果是无参构造函数，bean标签写了id和class属性即可。如果是有参数的构造函数，那么需要使用`<constructor-arg/>`标签：
```
<bean id="beanOne" class="x.y.ThingOne">
    <constructor-arg ref="beanTwo"/>
    <constructor-arg ref="beanThree"/>
</bean>

<bean id="beanTwo" class="x.y.ThingTwo"/>
<bean id="beanThree" class="x.y.ThingThree"/>
```
上面是通过参数的类型来解析，beanTwo和beanThree的值仍由spring的规则构造

如果是简单类型，例如int、String等，则如下：
```
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg type="int" value="7500000"/>
    <constructor-arg type="java.lang.String" value="42"/>
</bean>
```

还可以使用**参数索引位置**、**参数名称**等方式注入，这里省略

### 静态方法注入
如果类自身提供了静态方法可以用于提供实例，那么spring可以通过该静态方法获取实例。
```
public class ClientService {
    private static ClientService clientService = new ClientService();
    private ClientService() {}

    public static ClientService createInstance() {
        return clientService;
    }
}
```
xml配置：
```
<bean id="clientService"
    class="examples.ClientService"
    factory-method="createInstance"/>
```


### 实例工厂注入
spring支持使用一个工厂类来提供实例。先写出实例工厂类：
```
public class DefaultServiceLocator {

    private static ClientService clientService = new ClientServiceImpl();

    private static AccountService accountService = new AccountServiceImpl();

    public ClientService createClientServiceInstance() {
        return clientService;
    }

    public AccountService createAccountServiceInstance() {
        return accountService;
    }
}
```

然后在xml中配置：
```
<bean id="serviceLocator" class="examples.DefaultServiceLocator">
    <!-- inject any dependencies required by this locator bean -->
</bean>

<bean id="clientService"
    factory-bean="serviceLocator"
    factory-method="createClientServiceInstance"/>

<bean id="accountService"
    factory-bean="serviceLocator"
    factory-method="createAccountServiceInstance"/>
```

### Setter注入
使用Setter方式注入，则在bean中使用`<property/>`标签，这个方式在最开始的例子中已经展示过，


### 集合、null
如果值的类型为集合，或者需要赋值为null，方式如下：

首先定义类：
```
public class Student {
    private String name;
    private List<Integer> scores;
    private List<TestType> testTypeList;
    //... 省略getter和setter
}
```

然后配置xml：
```
<bean id="student" class="com.hfy.Student">
    <property name="name" value="颖颖颖"/>
    <property name="scores">
        <list>
            <value>1</value>
            <value>2</value>
        </list>
    </property>
    <property name="testTypeList">
        <list>
            <ref bean="aaa"></ref>
            <ref bean="aaa"></ref>
            <ref bean="bbb"></ref>
        </list>
    </property>
</bean>
<bean id="aaa" class="com.hfy.TestType"></bean>
<bean id="bbb" class="com.hfy.TestType"></bean>
```
int、String等基本类型可以直接写在value标签中，而其他的类型需要引用对应的bean。

> 其他集合类型的方式类似，可参考官方文档

### 内部Bean
<bean id="outer" class="...">
    <!-- 使用内部bean的方式，可以不使用ref，直接在内部声明 -->
    <property name="target">
        <bean class="com.example.Person"> <!-- this is the inner bean -->
            <property name="name" value="Fiona Apple"/>
            <property name="age" value="25"/>
        </bean>
    </property>
</bean>

### p-namespace和c-namespace
添加p或者c的命名空间，可以直接在bean标签的属性中设定相关值，而不用在子标签中写。p标签对应setter注入，c标签对应构造注入。使用方式简单，可到官方文档查看

--------------------------
> 依赖注入还有一些针对特殊场景的配置方式，例如depends-on、lazy-init。这些不在这里展开描述

## 4. Bean作用域
|作用域|描述|
|---|---|
|singleton|（默认）在每个IoC容器中，配置的一个bean只存在一个实例|
|prototype|每次获取都会产生新的实例|
|request|在一个http请求期间为单例。仅web开发中使用|
|session|在一个HTTP会话期间为单例。仅web开发中使用|
|application|在ServletContext生命周期中为单例。仅web开发中使用|
|websocket|在WebSocket的生命周期中为单例。仅web开发中使用|

除了`singleton`和`prototype`，其余的作用域仅在web开发中使用，也就是仅在使用基于web的spring的ApplicationContext时才有效（例如XmlWebApplicationContext），如果使用其他常规的IoC容器，例如ClassPathXmlApplicationContext，那么会抛出异常

## 5. Autowire 自动装配
xml配置中，组装bean可以不手动设置property或者其他标签，而是使用autowire属性就可以注入依赖。

声明一个Student类，依赖Pen类型
```
public class Student {
    private Pen pen;
    //...省略setter和getter
}
```
xml配置bean：
```
<bean id="student" class="com.hfy.Student" autowire="byName"/>
<bean id="pen" class="com.hfy.Pen"/>
```
autowire有几种选项，以下是常用的：
* byName：会自动寻找和类中的set方法后面的值相应id的bean，id必须唯一
* byType：根据类型寻找对应的bean，该class对应的bean必须唯一
* constructor：使用构造函数注入

## 6. 注解配置
Spring也可以通过注解来配置依赖关系。要使用注解，xml配置需要修改为：
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

</beans>
```

### @Autowired
除了在xml中使用autowire属性来自动装配，还可以使用注解的方式。@Autowired可以注解构造函数、setter方法、普通方法、字段。

Studen类，用@Autowired注解了pen字段：
```
public class Student {
    @Autowired
    private Pen pen;
}
```
xml配置如下：
```
<bean id="student" class="com.hfy.Student"/>
<bean id="pen" class="com.hfy.Pen"/>
```
此时student中不需要配置property，通过@Autowired就可以注入pen对象

@Autowired首先选择byType方式注入，当同类型的bean数量大于1时，就选择byName方式注入，否则异常。

> @Qualifier等注解，可以解决@Autowired有多个候选对象的情况

### @Component
使用注解，可以进一步得缩减xml配置，通过@Component注解，可以不用在xml中配置bean，@Component相当于bean配置的作用。要使用@Component注解，必须在xml中加上需要扫描@Component注解的包名：
```
<!--  加上这个配置，就会使com.hfy包中使用@Component的类生效  -->
<context:component-scan base-package="com.hfy"/>
```

在Student和Pen都使用@Component：
```
@Component
public class Student {
    @Autowired
    private Pen pen;
}

@Component
public class Pen {
    public void write() {
        System.out.println("写字");
    }
}
```
然后通过student这个id就可以获得Student实例，其中的pen也按此方式获得。

@Component可以加上名称参数，比如在Student类上使用@Component("test")，此时就需要通过test这个id来获得Student实例。

> 和`@Component`作用相同的还有`@Repository`、`@Service`和`@Controller`，提供了语义化的作用，适用于controller、service、dao的分层结构

### @Scope
@Scope用于配置作用域，可以用于类或者方法。

```
@Scope("prototype")
@Component
public class Student {
    //...
}
```
例如给Student类加上@Scope("prototype")，就会导致每次通过student获取的Student实例都是新建的。

### @Bean
注解方法，可以使该方法返回的实例和其类型作为bean，可以用于@Component和@Configuration中

## 7. 基于Java的容器配置
使用基于Java的容器配置，可以完全不使用xml，实现方式当然还是通过注解。

使用@Configuration注解类，表示该类作为配置类，并且也作为一个Component。这里故意既用@Autowired修饰一个pen字段，也用@Bean修饰一个创建Pen的方法。
```
@Configuration
@ComponentScan("com.hfy")
public class Student {
    @Autowired
    private Pen pen;

    @Bean
    public Pen newPen(){
        return new Pen();
    }
}

@Component
public class Pen {
    public void write() {
        System.out.println("写字");
    }
}
```
然后使用AnnotationConfigApplicationContext实例化容器，并获取Bean实例：
```
AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(Student.class);
Student student = (Student) applicationContext.getBean("student");
Pen pen = (Pen) applicationContext.getBean("newPen");
```
student中的pen字段直接从@Component修饰的Pen类构造而来，而getBean("newPen")则会通过newPen()方法获取pen实例

## 8. AOP编程
* `Aspect`（切面）： Aspect 作为一个声明，在 Aspect 中会包含着一些 Pointcut 以及相应的 Advice，也就是说表示了增强的操作，和在哪里插入增强。
* `Joint point`（连接点）：表示在程序中明确定义的点，典型的包括方法调用，对类成员的访问以及异常处理程序块的执行等等，它自身还可以嵌套其它 joint point。
* `Pointcut`（切点）：表示一组 joint point，这些 joint point 或是通过逻辑关系组合起来，或是通过通配、正则表达式等方式集中起来，它定义了相应的 Advice 将要发生的地方。
* `Advice`（增强）：Advice 定义了在 Pointcut 里面定义的程序点具体要做的操作，它通过 before、after 和 around 来区别是在每个 joint point 之前、之后还是代替执行的代码。
* `Target`（目标对象）：织入 Advice 的目标对象.。
*  `Weaving`（织入）：将 Aspect 和其他对象连接起来, 并创建 Adviced object 的过程

需要引入依赖：
```
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.9.6</version>
</dependency>
```

### 方式1 XML配置
通过XML配置，需要在xml中使用如下命名空间：
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">
</beans>
```
先定义切面类：
```
public class MyAspect {
    public void before() {
        System.out.println("执行前");
    }
}
```
接着配置xml：
```
<bean id="student" class="com.hfy.Student"/>
<bean id="myAspect" class="com.hfy.MyAspect"/>

<aop:config>
    <!-- 自定义切面，ref引用的bean中的方法可以用于在切入点中执行 -->
    <aop:aspect ref="myAspect">
        <!-- 定义一个切入点，在Student类的任意方法中切入 -->
        <aop:pointcut id="pointcut" expression="execution(* com.hfy.Student.*(..))"/>
        <!-- 在pointcut这个切入点执行前，执行beforeAdvice方法，beforeAdvice方法来自于MyAspect -->
        <aop:before method="beforeAdvice" pointcut-ref="pointcut"/>
    </aop:aspect>
</aop:config>
```

以上体现了一个基本的使用xml的aop配置，执行Student的任意成员方法时，都会先执行MyAspect的beforeAdvice方法

**除了使用`<aop:aspect>`的及其ref属性来配合`<aop:before>`等标签的方式声明增强（Advice）方法，还可以直接使用`<aop:advisor>`**，方式如下：

首先定义一个类，实现MethodBeforeAdvice或者AfterReturningAdvice之类的接口，也就是作为Advice：
```
public class MyAdvice implements MethodBeforeAdvice {

    public void before(Method method, Object[] objects, Object target) throws Throwable {
        System.out.println(target.getClass().getName() + "的" + method.getName() + "执行前");
    }
}
```
MyAdvice实现了MethodBeforeAdvice接口，before方法将在切入点之前执行。

```
<bean id="student" class="com.hfy.Student"/>
<bean id="myAdvice" class="com.hfy.MyAdvice"/>
<aop:config>
    <!-- 切入点为Student的所有成员方法 -->
    <aop:pointcut id="pointcut" expression="execution(* com.hfy.Student.*(..))"/>
    <!-- 在切入点中，执行myAdvice中的增强方法 -->
    <aop:advisor advice-ref="myAdvice" pointcut-ref="pointcut"/>
</aop:config>
```

### 方式2 注解
通过注解也可以实现AOP。

首先我们定义一个切面类：
```
@Aspect
public class MyAspect {
    @Before("execution(* com.hfy.Student.*(..))")
    public void beforeAdvice() {
        System.out.println("前置增强");
    }
}
```
@Aspect表示这个类是一个切面，而@Before注解的execution值将会使beforeAdvice()方法在Student的任意成员方法执行前执行。对应的切入点注解还有@Before、@After和@Around等。

然后在xml中配置：
```
<bean id="student" class="com.hfy.Student"/>
<bean id="myAspect" class="com.hfy.MyAspect"/>
<!--  开启aop的注解支持  -->
<aop:aspectj-autoproxy/>
```

这样就会在执行Student实例的方法前，先执行MyAspect的beforeAdvice()方法

前面定义切入点的方式可以认为是简写，也可以用下面这种方式来表示：
```
@Aspect
public class MyAspect {
    @Before("ponintcut()")
    public void beforeAdvice() {
        System.out.println("前置增强");
    }

    @Pointcut("execution(* com.hfy.Student.*(..))")
    public void ponintcut(){

    }
}
```
也就是说，使用@Pointcut的话，可以复用切入点

## 事务
@Transactional 可以注解方法、类。还需要配置数据源。