省略可以搜索到的各种描述，记录容易产生疑惑的问题。

Clean架构对应到Android应用软件，可以分为3层：
1. Data层
2. Domain层
3. Presentation层
按平时的理解来说，就是Presentation层依赖Domain层，Domain层依赖Data层，但是Clean架构中，Data层和Presentation层都属于外层，Domain层属于核心内层。

所以实际的依赖关系为：Presentation层依赖Data层和Domain层，Data层依赖Domain层。Presentation层使用Domain层的时候把Data层传给Domain层，也就是依赖倒置原则。Data层要实现Domain层的接口，例如Repository接口。

越内层越抽象，越外层越具体。

[可参考](https://www.jianshu.com/p/66e749e19f0d)

https://juejin.cn/post/6844904176296673287

## 测试方式
1. Data层：
2. Domain层
3. Presentation层

View  --  Presenter或ViewModel --  UserCase(包含线程调度，回调等)  --  Model(Repository实现接口DataSource，内部操作local和remote，local和remote也实现DataSource接口，DataSource接口包含读写数据的基本方法)

domain都是userCase

View向UserCase中注入Repository，依赖倒置

数据库表，是业务相关，应该属于domain的model，而data层只是数据库实现和dao

----

业务逻辑对外界实现完全没有依赖，任你外面的实现细节如何改变，核心业务逻辑不需修改。

依赖规则
为了实现上面所讲的特点，只需要遵循依赖规则：只允许外部圆层的代码依赖内部圆层的代码，反之则禁止。换言之，内部同心圆层的代码不知道任何外部同心圆层的代码，
比如内层的代码一律禁止引用外层声明的函数、类、变量或其他任何元素。

架与驱动（Frameworks and Drivers）层：这是最外面的一层，是很多工具（或库）的具体实现细节所在层，比如 Web 框架（考虑一个应用可以做成网页版应用，也可以做成单机版应用）、数据库工具包（考虑各种 ORM 的实现，以及各种数据库的驱动依赖包实现）等。

使用依赖注入实现控制反转