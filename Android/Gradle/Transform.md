## AGP Transform

* Android Gradle Plugin 1.5版本增加了Transform，我们可以自定义Transform用于处理class文件，修改字节码文件。
* Transform 为链式结构，每个 Transform 都是一个 Gradle 的 Task，自定义的Transform，会放到链式结构的最前面
* Transform 任务的输出会存储在 module/build/intermediates/transform/[Transform Name]/[Variant] 文件夹中。module为注册Transform的模块

### 自定义Transform

自定义的transform，需要注册到gradle的android扩展中。因为主要是展示Transform的用法，所以没有使用独立项目来开发Plugin，而是在buildSrc目录中编写Plugin和Transform

> 选择Kotlin，语法提示比groovy更友好

```kotlin
// buildSrc内的代码文件（假定是对于app模块使用的插件，所以要在app模块的build.gradle应用此插件）
class MyPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        project.extensions.getByType(AppExtension::class.java)
            .registerTransform(MyTransform(project))

        // 如果是groovy的话，可以简单点
        // project.extensions.getByName("android").registerTransform(MyTransform())
    }
}
```

然后，要自定义Transform，需要声明依赖（因为本文的示例是在buildSrc中写的，所以是在此文件声明依赖）

```groovy
// buildSrc/build.gradle文件
implementation("com.android.tools.build:gradle:7.1.0")
```

自定义Transform：

```kotlin
class MyTransform : Transform() {

    // 指定Transform的名称。生成的Transform任务的名称会用到 getName()、getInputTypes()以及Variant来拼接，可以在gradle任务列表中查看
    override fun getName(): String {
        return "MyTransform"
    }

    // 指定输入文件类型。
    // 我们只能使用DefaultContentType中的CLASSES、RESOURCES两种类型；ExtendedContentType中的类型只能AGP内置的Transform使用
    override fun getInputTypes(): MutableSet<QualifiedContent.ContentType> {
        return TransformManager.CONTENT_CLASS
    }

    // 指定输入文件范围。
    // PROJECT：仅当前Project
    // SUB_PROJECTS：仅子Projects（只能在app模块使用）
    // EXTERNAL_LIBRARIES：外部库，包括当前模块和子模块本地或远程依赖的 JAR/AAR（只能在app模块使用）
    // TESTED_CODE：当前variant测试的代码，包括依赖
    // PROVIDED_ONLY：provided-only的本地或远程依赖
    override fun getScopes(): MutableSet<in QualifiedContent.Scope> {
        return TransformManager.SCOPE_FULL_PROJECT
    }

    // 是否支持增量编译（并不完全取决于这个返回值）。如果是的话，TransformInput会包含changed/removed/added的文件列表
    override fun isIncremental(): Boolean {
        return false
    }

    // 输入信息被打包为TransformInvocation类型参数
    override fun transform(transformInvocation: TransformInvocation) {
        val outputProvider = transformInvocation.outputProvider

        transformInvocation.inputs.forEach { input ->
            input.jarInputs.forEach { jarInput ->
                //处理Jar
                transformJarInput(jarInput, outputProvider)
            }

            input.directoryInputs.forEach { directoryInput ->
                //处理源码文件
                transformDirectoryInputs(directoryInput, outputProvider)
            }
        }
    }

    private fun transformJarInput(jarInput: JarInput, outputProvider: TransformOutputProvider) {
        val dest = outputProvider.getContentLocation(
            jarInput.file.absolutePath,
            jarInput.contentTypes,
            jarInput.scopes,
            Format.JAR
        )
        //TODO 修改字节码（还需要处理jar文件，可能用到JarFile、JarEntry、ZipFile、ZipOutputStream、ZipInputStream等工具类）
        //将修改过的字节码copy到dest，就可以实现编译期间干预字节码的目的了
        FileUtils.copyFile(jarInput.file, dest)
    }

    private fun transformDirectoryInputs(
        directoryInput: DirectoryInput,
        outputProvider: TransformOutputProvider
    ) {
        transformDirectoryInputsByASM(directoryInput, outputProvider)

        val dest = outputProvider.getContentLocation(
            directoryInput.name,
            directoryInput.contentTypes, directoryInput.scopes,
            Format.DIRECTORY
        )
        //TODO 修改字节码
        //将修改过的字节码copy到dest
        FileUtils.copyDirectory(directoryInput.getFile(), dest)
    }

    // 还有一些有默认实现的函数，具体可在使用时再看看是否要用到。
    // ------------------------------------------------------------------------
    override fun applyToVariant(variant: VariantInfo?): Boolean {
        return super.applyToVariant(variant)
    }

    override fun getOutputTypes(): MutableSet<QualifiedContent.ContentType> {
        return super.getOutputTypes()
    }

    override fun getReferencedScopes(): MutableSet<in QualifiedContent.Scope> {
        return super.getReferencedScopes()
    }

    override fun getSecondaryFileInputs(): MutableCollection<File> {
        return super.getSecondaryFileInputs()
    }

    override fun getSecondaryFiles(): MutableCollection<SecondaryFile> {
        return super.getSecondaryFiles()
    }

    override fun getSecondaryFileOutputs(): MutableCollection<File> {
        return super.getSecondaryFileOutputs()
    }

    override fun getSecondaryDirectoryOutputs(): MutableCollection<File> {
        return super.getSecondaryDirectoryOutputs()
    }

    override fun getParameterInputs(): MutableMap<String, Any> {
        return super.getParameterInputs()
    }

    override fun setOutputDirectory(directory: Property<Directory>?) {
        super.setOutputDirectory(directory)
    }

    override fun setOutputFile(file: Property<RegularFile>?) {
        super.setOutputFile(file)
    }

    override fun isCacheable(): Boolean {
        return super.isCacheable()
    }
    // ------------------------------------------------------------------------
}
```

* getScopes()：这个范围的输入会被消费，所谓的消费就是被当前Transform使用，并且当前Transform必须把修改后的字节码文件复制到输出目录
* getReferencedScopes()：不会被消费，所以不用复制，但也不能修改。比如Instant run就只是判断一下字节码是否有变化，但并不用修改字节码文件。
* getSecondaryFiles()：额外的文件，这些文件发生变化时，也要触发Transform的执行，比如调试时修改Transform自身的代码，修改完后再执行也应该触发Transform

TransformInvocation的内容如下：

```java
public interface TransformInvocation {

    // 获取一些上下文信息
    @NonNull
    Context getContext();

    // 需要消费的输入内容
    @NonNull
    Collection<TransformInput> getInputs();

    // 不用消费的输入内容
    @NonNull Collection<TransformInput> getReferencedInputs();
    
    @NonNull Collection<SecondaryInput> getSecondaryInputs();

    // 用于得到输出的文件和目录
    @Nullable
    TransformOutputProvider getOutputProvider();

    // 当前Transform是否为增量执行
    boolean isIncremental();
}

public interface TransformOutputProvider {

    // 删除全部内容。在非增量时就可以使用它来删除原本的文件
    void deleteAll() throws IOException;

    // 根据参数信息，返回应该output的文件，我们修改后的字节码就应该放到这里。
    // 如果Format是JAR，返回的就是要创建的jar文件；如果是DIRECTORY，返回的就是output的目录
    @NonNull
    File getContentLocation(
            @NonNull String name,
            @NonNull Set<QualifiedContent.ContentType> types,
            @NonNull Set<? super QualifiedContent.Scope> scopes,
            @NonNull Format format);
}
```

`TransformInvocation getInputs()` 返回的是 `Collection<TransformInput>`，每个 `TransformInput` 可以获取 `Collection<JarInput>` 和 `Collection<DirectoryInput>`。

以AGP 7.1为例，遍历`Collection<DirectoryInput>`，以自己这里的实际情况为例，这个集合有两个DirectoryInput（具体项目可能不同，本质是一样的），它们的file目录分别是如下两种情况：

* kotlin代码对应的class文件放在目录：.../app/build/tmp/kotlin-classes/debug，此目录内以包名目录结构存放具体的class文件
* Java（包括生成的BuildConfig）、AIDL代码的class文件放在目录： .../app/build/intermediates/javac/debug/classes，也以包名结构存放class文件
可以看到通过这个路径访问class文件，还需要遍历内部具体包名目录的class文件，比如 .../app/build/tmp/kotlin-classes/debug/xx/xx/xx.class

每个 `TransformInput` 调用 `getJarInputs()` 会得到 `Collection<JarInput>`，每个 JarInput 的 file 都对应一个jar文件。

* 比如路径：~/.gradle/caches/modules-2/files-2.1/org.jetbrains.kotlinx/kotlinx-coroutines-android/1.6.1/4e61fcdcc508cbaa37c4a284a50205d7c7767e37/kotlinx-coroutines-android-1.6.1.jar

### jar的处理
使用 javassist 将依赖的jar包中的 androidx/fragment/app/FragmentActivity 的onCreate方法的首尾添加日志的示例：
```
private fun transformJarInput(jarInput: JarInput, outputProvider: TransformOutputProvider) {
    val dest = outputProvider.getContentLocation(
        jarInput.file.absolutePath,
        jarInput.contentTypes,
        jarInput.scopes,
        Format.JAR
    )
    // 获取jar源路径对应的JarFile
    val jarFile = JarFile(jarInput.file)
    // 创建目的jar路径的JarOutputStream
    val jarOutputStream = JarOutputStream(FileOutputStream(dest))
    val entries: Enumeration<JarEntry> = jarFile.entries()
    // 遍历原本jar中的entry，可能是class文件，也可能是properties、META-INF之类的资源文件
    for (jarEntry in entries) {
        if (jarEntry.name == "androidx/fragment/app/FragmentActivity.class") {
            // 找到需要修改的class文件
            // 使用javassist来修改该class文件
            val classPool = ClassPool.getDefault()
            classPool.appendClassPath(jarInput.file.absolutePath)
            classPool.appendClassPath(project.extensions.getByType(AppExtension::class.java).bootClasspath[0].toString())
            val ctClass = classPool.get("androidx.fragment.app.FragmentActivity")
            if (ctClass.isFrozen) {
                ctClass.defrost()
            }
            // 在onCreate方法首尾添加log日志打印
            val ctMethod = ctClass.getDeclaredMethod("onCreate")
            ctMethod.insertBefore("android.util.Log.e(\"hhhhh\",\"hhhhh\");")
            ctMethod.insertAfter("android.util.Log.e(\"hhhhh\",\"hhhhh\");")
            jarOutputStream.putNextEntry(JarEntry(jarEntry.name))
            // 将新的字节码对应的字节数组写入jar中
            jarOutputStream.write(ctClass.toBytecode())
            jarOutputStream.closeEntry()
            ctClass.detach()
        } else {
            // 其他文件就直接复制到目的jar文件中
            jarOutputStream.putNextEntry(JarEntry(jarEntry.name))
            val inputStream = jarFile.getInputStream(jarEntry)
            inputStream.copyTo(jarOutputStream)
            inputStream.close()
            jarOutputStream.closeEntry()
        }
    }
    // 省略了一些文件操作的try-catch
    jarOutputStream.close()
    jarFile.close()
}
```

### 增量构建模板

```kotlin
class MyyTransform : Transform() {
    // ...... 省略其他方法
    override fun isIncremental(): Boolean {
        return true
    }

    override fun transform(transformInvocation: TransformInvocation) {
        val isIncremental = transformInvocation.isIncremental
        val outputProvider = transformInvocation.outputProvider

        if (!isIncremental) {
            // 非增量编译，清空
            outputProvider.deleteAll()
        }

        transformInvocation.inputs.forEach { input ->
            input.jarInputs.forEach { jarInput ->
                //处理Jar
                val dest = outputProvider.getContentLocation(
                    jarInput.file.absolutePath,
                    jarInput.contentTypes,
                    jarInput.scopes,
                    Format.JAR
                )
                if (isIncremental) {
                    // 增量编译
                    transformJarInputWhenIncremental(jarInput, dest)
                } else {
                    // 非增量编译
                    transformJarInput(jarInput, dest)
                }
            }

            input.directoryInputs.forEach { directoryInput ->
                // 处理文件
                val dest = outputProvider.getContentLocation(
                    directoryInput.file.absolutePath,
                    directoryInput.contentTypes,
                    directoryInput.scopes,
                    Format.DIRECTORY
                )
                if (isIncremental) {
                    //处理增量编译
                    transformDirectoryInputWhenIncremental(directoryInput, dest)
                } else {
                    transformDirectoryInput(directoryInput, dest)
                }
            }
        }
    }

    private fun transformJarInputWhenIncremental(jarInput: JarInput, dest: File) {
        when (jarInput.status) {
            Status.NOTCHANGED -> {}
            Status.ADDED, Status.CHANGED -> {
                // 对于添加和修改的文件，执行transform
                transformJarInput(jarInput, dest)
            }
            Status.REMOVED -> {
                // 对于已经删除的输入，就删除它对应的输出文件
                FileUtils.forceDelete(dest)
            }
        }
    }

    private fun transformJarInput(jarInput: JarInput, dest: File) {
        //TODO do some transform
        // 将修改过的字节码copy到dest
        FileUtils.copyFile(jarInput.file, dest)
    }

    private fun transformDirectoryInputWhenIncremental(directoryInput: DirectoryInput, dest: File) {
        FileUtils.forceMkdir(dest)
        val srcDirPath = directoryInput.file.absolutePath
        val destDirPath = dest.absolutePath
        val fileStatusMap = directoryInput.changedFiles
        fileStatusMap.forEach { (inputFile, status) ->
            val destFilePath = inputFile.absolutePath.replace(srcDirPath, destDirPath)
            val destFile = File(destFilePath)
            when (status) {
                Status.NOTCHANGED -> {}
                Status.ADDED, Status.CHANGED -> {
                    FileUtils.touch(destFile)
                    transformFile(inputFile, destFile, srcDirPath)
                }
                Status.REMOVED -> {
                    FileUtils.forceDelete(destFile)
                }
            }
        }
    }

    private fun transformDirectoryInput(directoryInput: DirectoryInput, dest: File) {
        // TODO 修改字节码
        // 将修改过的字节码copy到dest
        FileUtils.copyDirectory(directoryInput.getFile(), dest)
    }

    private fun transformFile(inputFile: File, destFile: File, srcDirPath: String) {
        // TODO 修改字节码
        FileUtils.copyFile(inputFile, destFile)
    }
}
```

并发编译，WaitableExecutor

## javassist

[javassist官方文档](https://www.javassist.org/tutorial/tutorial.html)

javassist提供了源码级别和字节码级别的api，但侧重于源码级别的api。javassist的使用比较简单，可以用接近源码的方式修改字节码文件。

### 核心概念

* CtClass：表示一个Java类文件
* ClassPool：CtClass的hash表容器，它可以按需读取类文件来构造CtClass对象，并缓存对象用于后面再次访问。
* CtMethod：表示一个方法
* CtField：表示一个字段
* CtConstructor：表示一个构造函数

### 使用方式示例

1. 修改已有的class文件
```java
// 1. 得到一个ClassPool，静态方法getDefault()得到的是单例，也可以通过构造函数自己构造
ClassPool pool = ClassPool.getDefault();

// 有些情况可能需要添加类搜索路径
classPool.insertClassPath(...);
classPool.appendClassPath(...);

// 2. 通过ClassPool的get()方法来获取已存在的class文件对应CtClass对象
CtClass ctClass = pool.get("test.Rectangle");
// 设置父类
cc.setSuperclass(pool.get("test.Point"));

// 3. 通过CtClass获取CtField，并做一些修改
CtField ctField = ctClass.getField("xxx")
ctField.setName("newName")
ctField.setModifiers(Modifier.PRIVATE)

// 4. 操作CtConstructor
CtConstructor ctConstructor = ctClass.getConstructor("构造函数签名")
ctConstructor.setBody("代码")

// 或者获取全部构造函数去处理
ctClass.getConstructors()

// 5. 操作CtConstructor
val ctMethod = ctClass.getDeclaredMethod("方法名称")
// 在该方法内代码的开始和末尾添加代码
ctMethod.insertBefore("android.util.Log.e(\"hhhhh\",\"hhhhh\");")
ctMethod.insertAfter("android.util.Log.e(\"hhhhh\",\"hhhhh\");")

// 调用 writeFile 后才会把修改同步到实际的class文件中
ctClass.writeFile();

// ctClass.toBytecode()可以得到当前修改后的字节码数据（字节数组）
```

2. 创建新的class文件
```java
ClassPool pool = ClassPool.getDefault();
// 1. 创建CtClass。指定了包名，最后写入文件的时候，会自动创建对应的包文件目录。
// makeClass还有其他重载，使用时视情况选择更方便的，比如可以直接传一个类文件的输入流，就不用搜索路径，提升速度
CtClass ctClass = classPool.makeClass("com.xxx.MyClass")

// 2. 创建名为name的Field
CtField ctField = CtField(classPool.get("java.lang.String"), "name", ctClass)
// 设置private
ctField.modifiers = Modifier.PRIVATE
// 把Field添加的class中
ctClass.addField(ctField)
// 还可以设置初始值：ctClass.addField(ctField, "初始值")
// CtField还有一些其他的函数可以使用
// CtField.make()
// CtField.Initializer

// 3. 创建返回类型void，名为printName、无参数的方法
CtMethod ctMethod = CtMethod(CtClass.voidType, "printName", arrayOf<CtClass>(), ctClass)
ctMethod.modifiers = Modifier.PUBLIC
// 设置方法的代码
ctMethod.setBody("{System.out.println(this.name);}")
// 把方法添加到class
ctClass.addMethod(ctMethod)
// 4. 写入文件
ctClass.writeFile("/Users/bytedance/Downloads/hhhtest/src/main/java/")
// 从classPool中删除此ctClass缓存（看情况使用）
ctClass.detach()
```

生成的Java类如下：
```java
package com.xxx;

public class MyClass {
    private String name;

    public void printName() {
        System.out.println(this.name);
    }

    public MyClass() {
    }
}
```

3. 其他的常用api介绍
```java
// 使用当前线程的 context class loader加载类，得到Java的Class对象
Class clazz = cctClassc.toClass()；

// 创建CtMethod的工具类，比如创建抽象方法
CtNewMethod.abstractMethod

// 创建接口
classPool.makeInterface("com.xxx.MyInterface")；

// 当CtClass对象调用writeFile()、toClass()、toBytecode()转换到类文件，
// 会进入冻结状态，之后不能再修改CtClass（为了警告JVM不能重新加载类，不要修改已加载的类），不过可以解除冻结
ctClass.defrost()
// 从classPool中删除缓存。比如为了减少内存占用。调用之后不能再使用改CtClass，只能通过ClassPool重新构造一个CtClass
ctClass.detach()

// 它查找类文件会使用默认的系统搜索路径，可以自行另外设置
ClassPool.getDefault()
// 添加类搜索路径到最前面
classPool.insertClassPath(...)
// 添加类搜索路径到最后面
classPool.appendClassPath(...)

// $0, $1, $2, ...、$args 参数数组可以用在代码中用于访问参数，还有其他可以查阅文档
ctMethod.insertBefore("{ System.out.println($1); System.out.println($2); }");

// 修改ctMethod对应方法内部代码里面，将调用method2的代码修改为指定代码，替换的代码可以结合$开头的各种标识
// MethodCall、ConstructorCall、FieldAccess等都类似
ctMethod.instrument(object : ExprEditor() {
    override fun edit(m: MethodCall) {
        if (m.getClassName() == "xxx" && m.getMethodName() == "method2") {
            m.replace("替换的代码")
        }
    }
})
```

javassist在AGP Transform中的使用示例：
```kotlin
private fun transformDirectoryInputs(
    directoryInput: DirectoryInput,
    outputProvider: TransformOutputProvider
) {
    val dest = outputProvider.getContentLocation(
        directoryInput.name,
        directoryInput.contentTypes, directoryInput.scopes,
        Format.DIRECTORY
    )
    // 遍历class文件
    directoryInput.file.walk().forEach { file ->
        // 找到名为MainActivity的class文件（比较简单的写法，实际使用中需要考虑更加完善）
        if (file.name == "MainActivity.class") {
            val classPool = ClassPool.getDefault()
            // 添加此类到搜索路径
            classPool.appendClassPath(directoryInput.file.absolutePath)
            // 添加Android sdk的类搜索路径
            classPool.appendClassPath(project.extensions.getByType(AppExtension::class.java).bootClasspath[0].toString())
            val ctClass = classPool.get("com.example.myapplication.MainActivity")
            if (ctClass.isFrozen) {
                ctClass.defrost()
            }
            // 找到它的onCreate方法
            val ctMethod = ctClass.getDeclaredMethod("onCreate")
            // 在方法的首尾打印日志
            ctMethod.insertBefore("android.util.Log.e(\"hhhhh\",\"hhhhh\");")
            ctMethod.insertAfter("android.util.Log.e(\"hhhhh\",\"hhhhh\");")
            ctClass.writeFile(directoryInput.file.absolutePath)
        }
        FileUtils.copyDirectory(directoryInput.file, dest)
    }
}
```

## ASM
[ASM官方文档](https://asm.ow2.io/developer-guide.html)

ASM也是一个用于代码生成和转换Java字节码文件的库，ASM在创建class字节码的过程中，操纵的级别是JVM的字节码级别。

### CoreAPI
CoreAPI基于事件，每个事件表示class的一个元素，比如头信息、属性、方法、构造函数等，把元素生成对应的事件，将事件序列生成字节码文件。

1. 遍历类结构
```kotlin
// 1. 创建ClassReader，可以通过文件流、字节数组、类全限定名称三种方式（全限定名称需要注意是否在classpath中）
val classReader = ClassReader(file.inputStream())

// 2. 调用ClassReader的accept，传入ClassVisitor
classReader.accept(
    MyClassVisitor(Opcodes.ASM9),
    // 一些解析选项，比如SKIP_DEBUG可以忽略class文件中的局部遍历表、label、lineNumber等调试信息，
    // SKIP_CODE可以忽略方法中的代码，一些相关的visitxxx方法就不会被回调，其他可以在使用时具体了解
    ClassReader.SKIP_DEBUG or ClassReader.SKIP_FRAMES
)

// 3. 自定义的ClassVisitor
class MyClassVisitor : ClassVisitor {
    constructor(api: Int) : super(api)
    constructor(api: Int, classVisitor: ClassVisitor?) : super(api, classVisitor)

    // 访问类的头信息
    override fun visit(
        version: Int,
        access: Int,
        name: String?,
        signature: String?,
        superName: String?,
        interfaces: Array<out String>?
    ) {
        println("call asm visit $name")
    }

    // 访问类的某个注解信息
    override fun visitAnnotation(descriptor: String?, visible: Boolean): AnnotationVisitor {
        println("call visitAnnotation $descriptor")
        return super.visitAnnotation(descriptor, visible)
    }

    // 访问类的某个方法信息
    override fun visitMethod(
        access: Int,
        name: String?,
        descriptor: String?,
        signature: String?,
        exceptions: Array<out String>?
    ): MethodVisitor {
        return object : MethodVisitor(Opcodes.ASM9) {
            override fun visitCode() {
                println("call visitCode")
            }

            override fun visitMethodInsn(
                opcode: Int,
                owner: String?,
                name: String?,
                descriptor: String?,
                isInterface: Boolean
            ) {
                println("call visitMethodInsn")
            }

            override fun visitInsn(opcode: Int) {
                println("call visitInsn $opcode")
            }

            override fun visitEnd() {
                println("call visitMethod visitEnd")
            }

            override fun visitParameter(name: String?, access: Int) {
                println("call visitParameter $name")
            }

            override fun visitIntInsn(opcode: Int, operand: Int) {
                println("call visitIntInsn")
            }

            // ...... MethodVisitor还有其他方法可以重写
        }
    }

    override fun visitField(
        access: Int,
        name: String?,
        descriptor: String?,
        signature: String?,
        value: Any?
    ): FieldVisitor {
        println("call asm visitField $name   $signature")

        return object : FieldVisitor(Opcodes.ASM9) {
            override fun visitEnd() {
                println("call FieldVisitor visitEnd")
            }

            // FieldVisitor还有其他方法可以重写
        }
    }

    // ...... ClassVisitor还有其他方法可以重写
}
```
`ClassReader`对应读取一个class，调用accept会遍历该class文件的各个元素，对每个元素都会回调 `ClassVisitor` 的各个方法，比如visitField就是遍历到class的某个field，visitMethod就是遍历到某个方法，我们就可以通过自定义 `ClassVisitor` 并重写各个方法，就可以监听到遍历各个元素。

以 `visitMethod` 为例，需要返回一个 `MethodVisitor`，因为一个方法和方法内部的代码也有很多元素需要遍历，所以设计一个`MethodVisitor`：比如visitCode表示开始遍历方法，visitMethodInsn表示遍历到调用方法的指令，visitInsn表示遍历到一个无操作数的指令。

2. 创建class文件
如果要创建一个新的class文件，需要使用 `ClassWriter`，`ClassWriter`也是一个`ClassVisitor`的子类，此时就需要我们自己来触发各种visitxxx回调，`ClassWriter`在各种回调中记录信息，最终产生一个class文件：

```
// 1. 先创建一个ClassWriter
val classWriter = ClassWriter(0)
var fieldVisitor: FieldVisitor
var methodVisitor: MethodVisitor

// 2. 传入类的头信息
classWriter.visit(Opcodes.V11, Opcodes.ACC_PUBLIC or Opcodes.ACC_SUPER, "com/example/test/AsmNewClass", null, "java/lang/Object", null)

// 非必需的
classWriter.visitSource("AsmNewClass.java", null)

// 3. 通过 FieldVisitor 创建一个String类型的变量
kotlin.run {
    fieldVisitor = classWriter.visitField(Opcodes.ACC_PRIVATE, "name", "Ljava/lang/String;", null, null)
    fieldVisitor.visitEnd()
}

// 4. 通过 MethodVisitor 创建构造函数
kotlin.run {
    methodVisitor = classWriter.visitMethod(Opcodes.ACC_PUBLIC, "<init>", "()V", null, null)
    methodVisitor.visitCode()
    val label0: Label = Label()
    methodVisitor.visitLabel(label0)
    methodVisitor.visitLineNumber(3, label0)
    methodVisitor.visitVarInsn(Opcodes.ALOAD, 0)
    methodVisitor.visitMethodInsn(Opcodes.INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false)
    val label1: Label = Label()
    methodVisitor.visitLabel(label1)
    methodVisitor.visitLineNumber(4, label1)
    methodVisitor.visitVarInsn(Opcodes.ALOAD, 0)
    methodVisitor.visitLdcInsn("test")
    methodVisitor.visitFieldInsn(Opcodes.PUTFIELD, "com/example/test/AsmNewClass", "name", "Ljava/lang/String;")
    methodVisitor.visitInsn(Opcodes.RETURN)
    val label2: Label = Label()
    methodVisitor.visitLabel(label2)
    methodVisitor.visitLocalVariable("this", "Lcom/example/test/AsmNewClass;", null, label0, label2, 0)
    methodVisitor.visitMaxs(2, 1)
    methodVisitor.visitEnd()
}

// 5. 通过 MethodVisitor 创建名为 printName 的方法，打印字符串
kotlin.run {
    methodVisitor = classWriter.visitMethod(Opcodes.ACC_PUBLIC, "printName", "()V", null, null)
    methodVisitor.visitCode()
    val label0: Label = Label()
    methodVisitor.visitLabel(label0)
    methodVisitor.visitLineNumber(7, label0)
    methodVisitor.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;")
    methodVisitor.visitVarInsn(Opcodes.ALOAD, 0)
    methodVisitor.visitFieldInsn(Opcodes.GETFIELD, "com/example/test/AsmNewClass", "name", "Ljava/lang/String;")
    methodVisitor.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false)
    val label1: Label = Label()
    methodVisitor.visitLabel(label1)
    methodVisitor.visitLineNumber(8, label1)
    methodVisitor.visitInsn(Opcodes.RETURN)
    val label2: Label = Label()
    methodVisitor.visitLabel(label2)
    methodVisitor.visitLocalVariable("this", "Lcom/example/test/AsmNewClass;", null, label0, label2, 0)
    methodVisitor.visitMaxs(2, 1)
    methodVisitor.visitEnd()
}
classWriter.visitEnd()
// 6. 将字节数据写入class文件（需要先创建好目录）
File("~/com/example/test/AsmNewClass.class").writeBytes(classWriter.toByteArray())
```

产生的class文件反编译后如下：
```
package com.example.test;

public class AsmNewClass {
    private String name = "test";

    public AsmNewClass() {
    }

    public void printName() {
        System.out.println(this.name);
    }
}
```
可以看出来，在使用ClassReader读取class文件时，ClassReader会回调各种visitxxx方法，而在单纯的创建class文件时，需要自己去调用各种visitxxx方法，传入各种元素的信息。

3. 修改现有的class文件
示例为修改MainActivity的onCreate方法，在该方法代码的首尾各打印日志
```
// 1. 先对class创建对应的 ClassReader
val classReader = ClassReader(file.inputStream())
// 2. 创建 ClassWriter，传入classReader
val classWriter = ClassWriter(classReader, ClassWriter.COMPUTE_FRAMES)
// 3. 创建自定义的 ClassVisitor，传入 classWriter（代理模式）
val classVisitor = MyClassVisitor(Opcodes.ASM9, classWriter)
// 4. 调用classReader.accept，传入自定义的 MyClassVisitor 实例
classReader.accept(
    classVisitor,
    ClassReader.SKIP_DEBUG or ClassReader.SKIP_FRAMES
)
FileUtils.writeByteArrayToFile(file, classWriter.toByteArray())

// 自定义ClassVisitor，使用参数有ClassVisitor的构造函数，各个方法默认会使用这个代理
class MyClassVisitor : ClassVisitor {
    constructor(api: Int) : super(api)
    constructor(api: Int, classVisitor: ClassVisitor?) : super(api, classVisitor)

    // 因为需要修改方法，所以要重写visitMethod
    override fun visitMethod(
        access: Int,
        name: String?,
        descriptor: String?,
        signature: String?,
        exceptions: Array<out String>?
    ): MethodVisitor {
        val methodVisitor = super.visitMethod(access, name, descriptor, signature, exceptions)
        if (name == "onCreate") {
            // onCreate方法，则返回自定义的MethodVisitor
            return MyMethodVisitor(Opcodes.ASM9, methodVisitor)
        }
        // 其他方法，返回默认的 methodVisitor
        return methodVisitor
    }
}

class MyMethodVisitor : MethodVisitor {
    constructor(api: Int) : super(api)
    constructor(api: Int, methodVisitor: MethodVisitor?) : super(api, methodVisitor)

    // visitCode对应开始访问方法，也就是方法的开始位置
    override fun visitCode() {
        super.visitCode()
        mv.visitLdcInsn("hhhhh")
        mv.visitLdcInsn("onCreate: start")
        mv.visitMethodInsn(Opcodes.INVOKESTATIC, "android/util/Log", "e", "(Ljava/lang/String;Ljava/lang/String;)I", false)
        mv.visitInsn(Opcodes.POP)
    }

    // 对应无操作数的指令
    override fun visitInsn(opcode: Int) {
        // 方法结束，对应return或者抛出异常的情况
        if (opcode >= Opcodes.IRETURN && opcode <= Opcodes.RETURN || opcode == Opcodes.ATHROW) {
            // 添加代码
            mv.visitLdcInsn("hhhhh");
            mv.visitLdcInsn("onCreate: end");
            mv.visitMethodInsn(Opcodes.INVOKESTATIC, "android/util/Log", "e", "(Ljava/lang/String;Ljava/lang/String;)I", false)
            mv.visitInsn(Opcodes.POP)
            mv.visitInsn(Opcodes.RETURN)
        }
        super.visitInsn(opcode)
    }
}
```

修改后的代码如下：
```
......省略其他代码

    // onCreate的首尾加上了日志打印
    public void onCreate(Bundle bundle) {
        Log.e("hhhhh", "onCreate: start");
        super.onCreate(bundle);
        setContentView(R.layout.activity_main);
        Log.e("hhhhh", "onCreate: end");
    }

......
```
比如要删除某个方法，就在 visitMethod 中判断到是需要删除的方法就返回null，即该方法对应的各种指令都没有调用，也就去掉了该方法。

* ClassReader.SKIP_FRAMES：Stack Map Frames特性的主要目的是在字节码指令中跟踪局部变量表的类型、操作数的类型。此配置可以跳过Stack Map
* ClassReader.EXPAND_FRAMES：
* ClassWriter.COMPUTE_MAXS：自动计算 MAXSTACK 和 MAXLOCALS。修改方法，如果操作数栈最大值发生变化，需要调整 visitMaxs，但使用 COMPUTE_MAXS 可以帮我们完成。
* ClassWriter.COMPUTE_FRAMES：包含 COMPUTE_MAXS 的功能，并且会自动计算 stack map frames，Method.visitFrame()将不会被忽略。


> ASM提供了AdviceAdapter等封装类，可以简化一些常见情况的代码，具体使用时可以查看了解。

> ASM提供了ASMifier工具类，可以帮我们转换class文件为对应的asm语法指令，方面对字节码不熟悉的开发者使用。也有 ASM bytecode outline 之类的IDE插件可以使用
### Tree api
Tree api 基于对象，每个对象表示class的一部分，属性、方法、构造函数等，基于对象的api就是建立在基于事件的api之上的。Tree api就是执行完core api的流程，并把遍历过程中的全部信息记录下来，方便做一些各个信息相关联的操作，比如要对class添加一个方法，打印全部field，tree api就可以很便捷的拿到全部field信息，而core api则需要我们自己在遍历过程中手动去缓存。使用较简单，用到时看api文档即可。

core api更快，内存占用更少，tree api更方便

## Transform Action由Gradle提供
[Gradle TransformAction](https://docs.gradle.org/current/userguide/artifact_transforms.html)

Gradle在7.0版本新增的TransformAction会作为AGP中的Transform的替代品。AGP中定义了一些TransformAction的子类，TransformClassesWithAsmTask就是其中一个子类，它内部调用了 AsmClassVisitorFactory，我们可以通过AsmClassVisitorFactory来便捷地使用ASM处理class文件。

https://developer.android.com/build/releases/gradle-plugin-api-updates?hl=zh-cn

https://github.com/android/gradle-recipes