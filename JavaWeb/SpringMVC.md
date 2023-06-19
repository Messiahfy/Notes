https://docs.spring.io/spring-framework/reference/web/webmvc.html

## 概述
Spring MVC围绕前台控制器模式设计，核心的Servlet，即DispatcherServlet，为请求处理提供了一个共享算法，而实际工作则由可配置的委托组件来执行。这种模式非常灵活，支持多种工作流程。

DispatcherServlet和任何Servlet一样，需要通过使用Java配置或在web.xml中根据Servlet规范进行声明和映射。而DispatcherServlet则使用Spring配置来发现它在请求映射、视图解析、异常处理等方面所需要的委托组件。

## 基本使用
### 1. 新建web结构的maven项目，添加SpringMvc依赖
```
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>5.3.0</version>
    </dependency>
</dependencies>
```
### 2. 配置web.xml，注册DispatcherServlet（也可以用代码注册的方式）
```
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
    <servlet>
        <servlet-name>springmvc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!-- DispatcherServlet绑定Spring的配置文件 -->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:springmvc-servlet.xml</param-value>
        </init-param>
        <!-- 启动级别1 -->
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <!-- / 匹配所有请求，不包括jsp，/* 匹配所有请求，包括jsp       -->
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```

### 3. 添加spring mvc的xml配置文件到resources目录
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       https://www.springframework.org/schema/beans/spring-beans.xsd">
    <!--处理器映射器，根据Url映射到Bean的名称-->
    <bean class="org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping"/>
    <!--处理器适配器，通过映射的Bean名称找到对应的Controller-->
    <bean class="org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter"/>
    <!--视图解析器：Controller内部使用ModelAndView.setViewName跳转，这里通过name结合前缀后置找到jsp文件-->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver" id="internalResourceViewResolver">
        <!--前缀-->
        <property name="prefix" value="/WEB-INF/jsp/"/>
        <!--后缀-->
        <property name="suffix" value=".jsp"/>
    </bean>
    <!-- 注册下一步要编写的Controller为bean -->
    <bean id="/hello" class="com.hfy.controller.MyController"/>
</beans>
```

### 4. 编写Controller
可以实现Controller接口或者使用注解：
```
public class MyController implements Controller {
    public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
        ModelAndView mv = new ModelAndView();

        //业务代码
        String result = "helloSpringMVC";
        mv.addObject("msg", result);

        //视图跳转
        mv.setViewName("test");

        return mv;
    }
}
```
这样在访问 http://localhost:8080/上下文/hello 时，就会跳转到/WEB-INF/jsp/下的test.jsp页面

> 最后要注意的是，IDEA默认不会把依赖的jar包打包进项目的输出文件中，所以需要自己到Project Structure--Artifacts--Output Layout--Available Elements中把依赖库导入到WEB-INF/lib中

## 使用注解
### 1. 仍然需要spring mvc的xml配置文件
但是配置不一样：
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       https://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       https://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/mvc
       http://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <!-- 扫描com.hfy包中@Controller -->
    <context:component-scan base-package="com.hfy"/>
    <!-- 默认不处理静态资源-->
    <mvc:default-servlet-handler/>
    <!--mvc注解驱动，所以不再使用HandlerMapping和HandlerAdapter-->
    <mvc:annotation-driven/>
    <!--视图解析器：使用@Controller注解的类中，@RequestMapping注解的方法返回值作为视图的名称，名称结合前缀后置找到jsp文件-->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver" id="internalResourceViewResolver">
        <!--前缀-->
        <property name="prefix" value="/WEB-INF/jsp/"/>
        <!--后缀-->
        <property name="suffix" value=".jsp"/>
    </bean>
</beans>
```
然后声明Controller类：
```
@Controller
public class MyController {

    @RequestMapping("/test")
    public String hello(Model model) {
        model.addAttribute("msg", "Hello SpringMVC");
        return "test";
    }
}
```
用了 @RequestMapping("/test") 后，访问http://localhost:8080/上下文/test 就可以执行hello方法，然后通过返回的test去找到/WEB-INF/jsp/test.jsp

类上面也可以用@RequestMapping，如果在类上面使用@RequestMapping("/jxy")，那么访问就要使用http://localhost:8080/上下文/jxy/test

> Model用于向请求域共享数据，在视图解析器生成html页面的时候读取；但前后端分离的话就很少使用了。

## 获取参数
### GET请求
#### 1.直接声明参数
GET请求如果要接收参数，第一种方式如下，直接在方法中声明参数，参数名称就是GET请求查询参数名称：
```
@Controller
public class MyController {

    @RequestMapping("/test")
    public String hello(String name, Model model) {
        model.addAttribute("msg", "Hello SpringMVC");
        return "test";
    }
}
```
此时访问，http://localhost:8080/上下文/test?name=hfy，就可以得到HTTP请求的查询参数

参数也可以声明一个对象，然后查询参数要对象该对象的各个字段

####  2. 使用@requestParam
```
@Controller
public class MyController {

    @RequestMapping("/test")
    public String hello(@RequestParam(name = "id", required = true) String name, Model model) {
        model.addAttribute("msg", "Hello SpringMVC");
        return "test";
    }
}
```
访问方式：http://localhost:8080/springmvc_war_exploded/test?id=hfy，如果required为false，则可以不传，并得到默认值

#### 3.  使用@PathVariable
使用@PathVariable可以将参数直接作为path
```
@Controller
public class MyController {

    @RequestMapping("/test/{name}")
    public String hello(@PathVariable String name, Model model) {
        System.out.println(name);
        model.addAttribute("msg", "Hello SpringMVC");
        return "test";
    }
}
```
访问方式：http://localhost:8080/springmvc_war_exploded/test/hfy

#### 4. 使用HttpServletRequest
在方法的请求参数中加上HttpServletRequest
```
@Controller
public class MyController {

    @RequestMapping("/test")
    public String hello(HttpServletRequest request, Model model) {
        System.out.println(request.getParameter("name"));
        model.addAttribute("msg", "Hello SpringMVC");
        return "test";
    }
}
```
http://localhost:8080/springmvc_war_exploded/test?name=hfy

### POST请求
不详细展开，直接列举部分方式：
* 可以直接转为对象
* @RequestParam 也可以用于获得提交表单中的数据
* @RequestBody 将http请求体的数据转为java对象
* HttpServletRequest


----------------
* @RequestMapping用于所有HTTP请求方法，具体的有@GetMapping、@PostMapping等注解可以使用，注解的produces参数，可以指定返回的MIME类型，比如json
* RequestEntity参数，则能得到请求数据，使用RequestEntity.getHeaders()、RequestEntity.getBody()等。

https://www.cnblogs.com/throwable/p/11980555.html

### 重定向和跳转
可以用ModelAndView，包含了Model和设置视图的功能。使用Model、Map和ModelMap参数，实际传来的都是BindingAwareModelMap

也可以拿到HttpServletResponse，不用视图解析器

## 前后端分离
前面的例子，用到视图解析器的话，就还是传统的前后端一体的方式，Controller返回的数据都对应页面。而现在前后端分离的情况，则返回的数据并不是一个页面，而是在响应体中携带数据。

可以使用HttpServletResponse，直接设置响应体
```
@RequestMapping("/test")
public void hello(HttpServletResponse response) throws IOException{
    response.getWriter().print("hello");
}
```

实际开发中，更多会使用@ResponseBody
```
@RequestMapping("/test")
@ResponseBody
public String hello() {
    return "hahaha";
}
```
使用了@ResponseBody，此时返回的数据不会再被视图解析器处理，而是直接放到http响应体中。

也可以返回对象，并自动转换为json放到响应体中（需要配置json处理）：
```
@RequestMapping("/test")
@ResponseBody
public User hello() {
    return new User();
}
```
也就对应了最常见的移动端访问的方式。

如果用@RestController注解类，就等于同时使用@Controller和@ResponseBody

> 返回值使用 ResponseEntity ，表示返回的响应报文

## 拦截器
SpringMVC的拦截器作用于DispatcherServlet和Controller之间

## 异常处理器
HandlerExceptionResolver，SpringMVC默认使用 DefaultHandlerExceptionResolver

## 完全代码注解配置SpringMVC
完全注解，替代web.xml，使用AbstractAnnotationConfigDispatcherServletInitializer

## SpringMVC的执行流程
### 常用组件


### DispatcherServlet
* 初始化：DispatcherServlet继承自Servlet，它的初始化过程中可以看到重要的功能，比如 initHandlerMappings() 初始化Controller。
* 处理请求：调用组件处理请求，比如重写了doGet、doPost等，都会调用processRequest-->doService-->doDispatch