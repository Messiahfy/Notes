#### 1.概述
Path和Files属于NIO中的类，主要用法就是Path表示一个文件或目录的路径，然后用Files类的静态方法来操作。

#### 获取Path的方式
Path absolute = Paths.get("/home","huang");
Path relative = Paths.get("myprog","conf","user.properties");

Path path = file.toPath();

#### 用Files的静态方法操作文件
###### 读入（写出类似）
byte[] bytes = Files.readAllBytes(path);//读取为字节数组

List<String> = Files.readAllLines(path, charset);//按行序列读入字符串

###### 创建文件和目录
创建目录的两种方法，不同的是如果中间目录不存在，下面的会都创建。比如/home/huang/abc，如果/huang不存在，方法1会抛出错误，方法2会把/huang和/path都创建。
`Files.createDirectory(path)`
`Files.createDirectories(path)`

创建文件
`Files.createFile(path)`

**其他方法参考API文档**

