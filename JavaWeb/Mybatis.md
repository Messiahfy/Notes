## 使用步骤
这里以一个简单的查询表的全部数据为例。

[官方文档](https://mybatis.org/mybatis-3/zh/index.html)
### 1.添加maven依赖
在pom.xml的dependencies标签内，加上mybatis和对应的mysql连接器的库
```
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.6</version>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.22</version>
</dependency>
```

### 2.使用XML构建 SqlSessionFactory
在maven项目的resources目录中添加mybatis-config.xml文件：
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306?mybatis"/>
                <property name="username" value="root"/>
                <property name="password" value=""/>
            </dataSource>
        </environment>
    </environments>
</configuration>
```
这个xml文件用于配置Mybatis的环境，通过如下方式读取并得到 SqlSessionFactory：
```
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

> maven项目结构中的resources目录中的文件，会被maven直接复制到目标文件的根目录

### 3.创建数据结构和Dao层代码
创建数据表对应的数据类：
```
public class User {
    private int id;
    private String name;
    private String pwd;
    ... 省略getter、setter等
}
```

创建Dao层接口：
```
public interface UserDao {
    List<User> getUserList();

    User getUserById(int id);
}
```

### 4.创建接口函数和sql语句的映射
创建UserMapper.xml文件，放在resources目录中。如果想放在相关的Java文件目录，那么需要修改maven的配置，否则不会把该xml文件放到目标目录中。
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.hfy.dao.UserDao">
    <select id="getUserList" resultType="com.hfy.pojo.User">select * from mybatis.user</select>
    <select id="getUserById" parameterType="int"
            resultType="com.hfy.pojo.User">select * from mybatis.user where id =#{id}</select>
</mapper>
```
namespace对应接口的全限定名称，select的id属性对应UserDao中的方法名，parameterType属性对应参数类型，resultType属性对应返回的类型，标签的内容对应sql语句

> 也可以使用注解的方式来关联方法和sql语句，这样在下一步只需要用class属性绑定类，而不用resource属性。但对于稍微复杂一点的语句，仍建议使用xml方式

### 5.在mybatis-config.xml文件中加上映射配置
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        ...
    </environments>
    <mappers>
        <mapper resource="UserMapper.xml"/>
    </mappers>
</configuration>
```

### 6.执行查询
```
SqlSession sqlSession = sqlSessionFactory.openSession();
// 方式1
UserDao userDao = sqlSession.getMapper(UserDao.class);
List<User> userList = userDao.getUserList();
User user = userDao.getUserById(1);

// 方式2
//List<User> userList = sqlSession.selectList("com.hfy.dao.UserDao.getUserList");

sqlSession.close();
```
两种方式，都会根据映射关系，执行配置的sql语句。

## 结果映射、动态sql、缓存等功能
复杂查询时需要用到ResultMap，拼接sql时需要用到动态sql，这些部分内容可以查看官方文档即可。