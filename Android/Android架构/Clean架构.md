省略可以搜索到的各种描述，记录容易产生疑惑的问题。

Clean架构对应到Android应用软件，可以分为3层：
1. Data层
2. Domain层
3. Presentation层
按平时的理解来说，就是Presentation层依赖Domain层，Domain层依赖Data层，但是Clean架构中，Data层和Presentation层都属于外层，Domain层属于核心内层。

所以实际的依赖关系为：Presentation层依赖Data层和Domain层，Data层依赖Domain层。Presentation层使用Domain层的时候把Data层传给Domain层，也就是依赖倒置原则。Data层要实现Domain层的接口，例如Repository接口。

越内层越抽象，越外层越具体。

[可参考](https://www.jianshu.com/p/66e749e19f0d)

## 测试方式
1. Data层：
2. Domain层
3. Presentation层