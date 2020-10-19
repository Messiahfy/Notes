## 埋点的一些常用方式
1. 手动埋点
2. AccessibilityDelegate:遍历全部view设置监听
3. AppCompatDelegate：重写AppCompatActivity的getDelegate方法，返回我们的AppCompatDelegate，代理其中创建替换View的流程，可以替换View。
4. 使用Gradle Plugin:在编译期间修改class字节码，需要使用AspectJ、Javassist、ASM这些工具实现代码插桩

## ASM
直接操作字节码，可以生成字节码文件，也可以修改已有的字节码文件。偏向于底层

## AspectJ
AspectJ通过特定的语法，和自己的编译器来生成插入自定义逻辑的代码。

## Javassist
在源码层次操作字节码，使用示例：
```
public class JavassistTest {
    public static void main(String[] args) throws NotFoundException, CannotCompileException, IllegalAccessException, InstantiationException, IOException {
        ClassPool cp = ClassPool.getDefault();
        CtClass cc = cp.get("className");
        CtMethod m = cc.getDeclaredMethod("process");
        m.insertBefore("{ System.out.println(\"start\"); }");
        m.insertAfter("{ System.out.println(\"end\"); }");
        Class c = cc.toClass();
        cc.writeFile("~/project/");
        Base h = (Base)c.newInstance();
        h.process();
    }
}
```