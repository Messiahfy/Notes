## Apache Tomcat

Apache Tomcat是一个Web容器，实现了Java Servlet, JavaServer Pages, Java Expression Language 和 Java WebSocket技术。

Tomcat服务器的目录结构：  
/bin         存放windows和*nix平台的命令文件，如启动和停止Tomcat  
/conf        存放Tomcat服务器的各种配置文件  
/lib         存放Tomcat服务器所需的各种jar文件  
/logs        存放Tomcat的日志文件  
/temp        Tomcat运行时用于存放临时文件  
/webapps     当发布Web应用时，默认将Web应用文件发布至此目录。主动将Web应用文件放至此处，启动Tomcat即可访问。  
/work        Tomcat把由JSP生成的Servlet放于此目录  

### 部署项目
1. 示例：直接在/webapps中创建test目录，然后在test里面放一个test.html，那么就可以通过ip:端口/test/test.html来访问资源
2. 或者把资源打包成war，放到/webapps中，tomcat会自动解压
3. 配置conf/server.xml，在Host标签内加上 <Context docBase="项目存放的路径" path="虚拟目录" />
4. 在conf/catalina/localhost创建一个任意名称的xml，在xml中写 <Context docBase="项目存放的路径" /> path就是xml的文件名

### Web应用文件基本目录
WebApp Root/WEB-INF/web.xml 是部署描述文件  
WebApp Root/WEB-INF/classes 用来放置应用程序用到的自定义类(.class)，必须包括包(package)结构  
WebApp Root/WEB-INF/lib 用来放置应用程序用到的JAR文件  
WebApp Root/ 根目录下可存放可访问文件，如html，jsp等

Servlet项目，按这种规定的项目结构，打war包放到tomcat运行。（Servlet 3.0 已经可以用注解替代web.xml）

### IDEA部署
IDEA会给每个tomcat部署的项目单独创建配置文件

Project Strucure中的Artifacts的Type默认为Web Application：Exploded，这种情况IDEA在启动tomcat时会使用-Dcatalina.base=某个路径来设定临时的配置路径，在这个路径里面的conf/localhost里面配置http访问的虚拟路径和项目包的实际目录。可以在Run/Edit Configurations的Tomcat的Deployment中修改Application context来修改虚拟路径。

如果使用Web Application：Archive，这种方式才会把项目产出包部署到Tomcat的默认部署位置webapps。