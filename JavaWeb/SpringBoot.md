## 自动配置原理
@SpringBootApplication
    @SpringBootConfiguration  @Configuration  @Component
    @EnableAutoConfiguration  自动导入包
        @AutoConfigurationPackage  @Import({Registrar.class})自动注册包
        @Import({AutoConfigurationImportSelector.class}) 自动导入包的核心
            AutoConfigurationImportSelector#getCandidateConfigurations 加载EnableAutoConfiguration相关配置
                SpringFactoriesLoader#loadSpringFactories 这里将会找到spring-boot-autoconfigure包的META-INF/spring.factories文件
    @ComponentScan 扫描包

spring.factories文件中，可以找到EnableAutoConfiguration相关配置，包含了各种自动配置类，包括用于SpringMVC的WebMvcAutoConfiguration。每个自动配置类的@ConditionalOnClass注解中的参数，也就是一些类存在的情况，该自动配置类才会生效。

@SpringBootApplication注解的入口类，会执行SpringApplication的run方法，SpringApplication类主要完成下面四步：
1. 判断项目是普通项目还是Web项目
2. 查找和加载所有可用的初始化器，放到initializers属性中
3. 查找所有的应用程序监听器，放到listeners属性中
4. 推断并设置main方法的定义类，找到运行的主类（run方法会把当前主类设置到SpringApplication的primarySources属性，用于其他步骤使用）

## 配置方式
resources目录中，application.properties文件，也可以是yml、yaml格式，可以配置例如端口号等

还可以结合@ConfigurationProperties给@Component的类的属性注入值

可配置的选项，和每个@ConfigurationProperties对应

## 静态资源
静态资源放的目录，可以在WebMvcAutoConfiguration类中的addResourceHandlers方法中找到。

### 首页
WebMvcAutoConfiguration类中的getIndexHtml方法中可以看到放首页的方式，默认为index.html。也可以通过Controller的方式动态跳转。

## 自定义配置
使用@Configuration注解实现WebMvcConfigurer接口的类，重写各种方法即可。

练习可以参考 https://www.bilibili.com/video/BV1PE411i7CV?p=20