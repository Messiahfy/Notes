## 构成
* lib：so文件
* res：图片、布局、动画、其他
* assets
* dex：代码
* resources.arsc：资源索引
* 其他：Manifest、META-INF等

## 思路
能删就删，能缩减就缩减，能动态获取就动态获取

* 代码优化：字节码和dex优化（byteX去除R文件内联等操作、dex代码精简、常量内联，短方法内联）、R文件删除、混淆优化、使用D8、R8
* 资源优化：删除冗余、相似的资源、图片优化（压缩，或者使用矢量图，优先顺序svg webp png jpg，一般可以只要xhpi）、资源整体优化、指定资源配置，例如中文、shrinkResources和minifyEnable配置
* so优化：压缩、符号表缩减、临时下载
* 业务优化：减少不必要的第三方库、基础库统一、移除无用功能、插件化