## 1. Jnit4 常用注解

[Junit4 文档](https://junit.org/junit4/)

https://www.jianshu.com/p/3822a81b6cb0?utm_campaign=hugo&utm_medium=reader_share&utm_content=note

Android Studio中可以使用 shift+cmd+t 或者 navigate->Test 来对类或方法创建测试

| 注解 | 作用 |
|---|---|
| @Test | 表示此方法为测试方法。此注解还可以设置expected和timeout |
| @Before | 这个方法在每个测试之前执行，可用于初始化准备环境 |
| @After | 这个方法在每个测试之后执行，可用于清理环境 |
| @BeforeClass | 这个方法在所有测试开始之前执行一次，必须是static |
| @AfterClass | 这个方法在所有测试结束之后执行一次，必须是static |
| @Ignore | 忽略某个测试方法 |
| @RunWith | 注解在测试类上，指定该测试类使用某个运行器 |
| @Parameters | 指定测试类的测试数据集合，需要配合对应的Runner |
| @Rule | 注解自定义的rule，可以插入自定义的行为 | 

Assert类  MatcherAssert类

测试执行顺序 @FixMethodOrder

test runner

rule  ExpectedException

Suite测试 https://juejin.cn/post/6844903791494447112#heading-10
```
import org.junit.runner.RunWith;
import org.junit.runners.Suite;

@RunWith(Suite.class)
@Suite.SuiteClasses({
  TestFeatureLogin.class,
  TestFeatureLogout.class,
  TestFeatureNavigate.class,
  TestFeatureUpdate.class
})

public class FeatureTestSuite {
  // the class remains empty,
  // used only as a holder for the above annotations
}
```

## 执行流程
JUnit从JUnitCore的run方法开始执行，Request的classes方法将得到对应的Runner，对于JUnit4默认为BlockJUnit4ClassRunner，可以通过注解@RunWith来指定。

AndroidJUnit4ClassRunner

## 2. Parameters使用示例
```
@RunWith(Parameterized.class)
public class ParameterizedTest {

    private int num;
    private boolean odd;

    public ParameterizedTest(int num, boolean odd) {
        this.num = num;
        this.odd = odd;
    }

    // 被@Parameterized.Parameters注解的方法会把返回的列表数据中的元素对应注入
    // 到测试类的构造函数ParameterizedTest(int num, boolean odd)
    @Parameterized.Parameters
    public static Collection<Object[]> providerTruth() {
        return Arrays.asList(new Object[][]{
                {1, true},
                {3, true},
                {4, false},
                {111, true},
                {5, true},
                {-1, true}
        });
    }

//    //也可不使用构造函数注入的方式，而是注解注入public变量的方式
//    @Parameterized.Parameter
//    public int num;
//    //value = 1表示括号里的第二个Boolean值
//    @Parameterized.Parameter(value = 1)
//    public boolean odd;

    @Test
    public void printTest() {
        Assert.assertEquals(odd, print(num));
        System.out.println(num);
    }

    private boolean print(int num) {
        return num % 2 != 0;
    }
}
```