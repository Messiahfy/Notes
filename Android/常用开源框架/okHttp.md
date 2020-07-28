 从OkHttp的3.13.0版本开始，需要开启Java 8（可在`https://square.github.io/okhttp/changelog_3x/`中查看），retrofit从2.7.0开始就捆绑使用OkHttp的3.14.4版本，如果不启用Java 8，就要使用更低的版本

post 常见请求Content-Type
* application/x-www-form-urlencoded  提交表单，对应普通键值对
* application/json  json数据
* multipart/form-data  提交表单，包含多种类型，比如图片、文件

构造OkhttpClient，构造reque st（可能包含header、requestBody）

拦截器：每个拦截器在intercept方法中通过chain参数获取到request，将request修改或者自行创建新的request等任意处理方式，再调用chain.proceed(request)让下一级拦截器继续处理，流程就是递归调用再递归返回。