## Servlet的部署
1. 创建一个Tomcat项目后，在src里面创建实现Servlet接口的子类MyServlet
2. 在WEB-INF/web.xml里面配置：  
    ```
    <servlet>
        <servlet-name>test</servlet-name>
        <servlet-class>com.hfy.MyServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>test</servlet-name>
        <url-pattern>/test</url-pattern>
    </servlet-mapping>
    ```
现在就可以通过localhost:8080/test来访问这个MyServlet。（如果配置了虚拟路径，则test前面需要加上。例如使用IDEA的情况，会使用配置的Application context，也就是临时的catalina目录里配置的虚拟路径）

## 执行原理
1. url中写了localhost:8080/test
2. tomcat就会到web.xml里面找到/test对应的servlet
3. 把该servlet类的字节码加载到内存，创建该类的实例
4. 调用servlet实例的方法（生命周期）

## Servlet的生命周期
servlet是单例的
1. init：初始化，只执行一次（在web.xml中可以配置第一次访问时初始化还是服务器启动就初始化）
2. service：提供服务，每次访问都会执行
3. destroy：正常关闭服务器时，销毁之前执行
4. getServletConfig：
5. getServletInfo：

## 注解配置
Servlet 3.0开始，支持注解配置，不需要web.xml：
```
@WebServlet(urlPatterns = "/test")
public class MyServlet implements Servlet {
    ...
}
```
urlPatterns可以是数组，也就是可以匹配多个路径

## Servlet的实现类
javax中已经提供了一些Servlet接口的实现类
1. GenericServlet：对各个方法提供了空实现
2. HttpServlet：重写了service方法，提供了doGet、doPost等快捷方法，我们继承HttpServlet，然后重写doGet等方法即可

## Request和Response
service方法中，tomcat给我们传入了request和response，实际就是对应http的请求和响应。

### Request相关功能
#### HTTP数据相关功能
1. 获取请求的起始行数据
2. 获取请求的头部数据
3. 获取请求的body数据

例如头部数据，如果当前url由其他url跳转而来，则头部包含referer，值为之前的url，用这个可以防盗链，也就是要访问此url，必须从指定的网址跳转而来，而不能任意访问。

#### 请求转发
在服务器内部转发请求，比如把AServlet的请求转发盗BServlet中，使用方式：
1. 首先使用RequestDispatcher requestDispatcher=servletRequest.getRequestDispatcher("另一个servlet的访问路径");
2. 调用requestDispatcher.forward(servletRequest,servletResponse);

#### 数据共享
可以通过setAttribute相关方法在请求转发的多个servlet间传递数据

### Response相关功能
1. 设置响应的起始行
2. 设置响应的头部
3. 设置响应的body
4. 重定向

#### ServletContext
代表整个web应用，可以和web容器通信，可以通过request或者servlet获取，以下是常用的功能：
1. 获取MIME类型
2. 共享数据
3. 获取项目内文件在服务器的真实路径

## Cookie和Session
### 会话
一次会话包含多次请求和响应，通过会话技术可以在同一次会话的多次请求间共享数据，会话分为两种：
1. 客户端会话：Cookie
2. 服务器会话：Session

### Cookie
使用步骤：
1. 服务器创建Cookie：new Cookie(string,string)
2. 服务器发送Cookie给浏览器：response.addCookie(cookie)，可以添加多个
3. 浏览器下次请求会带上Cookie，服务器从request中获取：request.getCookies()

* cookie的数据放在HTTP的头部 set-cookie 中，浏览器会存储cookie
* cookie的有效时间，可以通过Cookie.setMaxAge(int seconds)来控制
* cookie默认只在当前web项目内共享，如果要在同服务器的多个项目共享，可以通过cookie.setPath("/")的方式；如果要在不同服务器间共享，需要使用setDomain("domain")，相同域名的不同服务器可以共享。

### Session
数据保存在服务器，在一次会话中，可以在多次请求中共享，使用步骤：
1. 获取HttpSession：request.getSession
2. 使用HttpSession：setAttribute等方法

Session的实现依赖于Cookie，第一次获取Session，会在内存创建一个Session对象，本次请求响应时，会设置Cookie，第二次请求，浏览器会带上该Cookie，服务器再获取Session，根据Cookie会查到之前内存中的Session。

用于Session的Cookie的生命周期在浏览器关闭后就会结束，如果希望下次打开浏览器发起请求，服务器还能获取同样的Session对象，那么可以手动设置Key为JSESSIONID的Cookie，并设置MaxAge。

tomcat在关闭服务器时，会序列化Session，再次打开会反序列化。

## Filter
访问服务器的资源时，过滤器可以拦截请求，增加一些通用功能，比如登录认证、编码设置等

1. 实现Filter接口
2. 配置，可以用注解或者xml

### 过滤器链
过滤器可以存在多个，注解方式，执行顺序按字符串比较规则；xml配置时，按配置先后顺序执行。

A过滤前 B过滤前 访问资源并响应 B过滤后 A过滤后

### 配置
配置中可以使用dispatcherTypes来控制浏览器直接访问才拦截，或者转发才拦截，还有其他情况，可以配置多个。

## Listenr
ServletContextListener，可以监听ServletContext的创建和销毁