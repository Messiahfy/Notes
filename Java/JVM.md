## 类加载过程
读取Class文件，将它转换为对应的静态数据结构存储在方法区，并在堆中生成一个便于用于调用的java.lang.Class类型的对象

Class文件可以来源于网络、本地文件、二进制流。。。

1. 加载
2. 验证：文件格式、元数据、字节码验证、符号引用验证。
3. 准备：给静态变量赋默认值
4. 解析：符号引用转为直接引用。比如A和B两个Java文件，编译为两个单独的class文件，A引用B只能是一个字符串来表示，而到运行时，加载了A，发现B还没有加载，就会去加载B，并且把引用改为内存中的地址（静态解析）。不过，如果是抽象类或者接口，就要在运行时再去修改为内存地址（动态解析）。
5. 初始化：也就是对象的构造过程，先申请内存，再调用初始化函数，最后让变量指向该内存
6. gc

### 类加载器双亲委派
1. Bootstrap ClassLoader：加载jre/lib目录中的类。启动类加载器不能被Java程序直接引用，如果要让它加载类就使用null来表示它（比如String的）。
2. Extension ClassLoader：加载jre/lib/ext目录的类
3. Application ClassLoader：加载classpath中的类，`ClassLoader.getSystemClassLoader()`也称为系统加载器，默认的
4. 自定义类加载器：可以去加载自己指定位置的类

类加载器内部都有一个缓存，记录已加载的类 缓存在native，避免重复加载。

双亲委派的主要设计目的是为了安全，比如自己写了一个java.lang.Object类，双亲委派模型的流程会让Bootstrap ClassLoader加载java自带的。

双亲委派模型可以破坏：越基础的类由越上层的ClassLoader进行加载，但如果基础类需要加载非基础的类，那就需要上层的类加载器去调用下层的类加载器，比如Thread类的ContextClassLoader就是为了这个目的设计的，上层类加载器取出给Thread设置的ContextClassLoader，让这个ContextClassLoader去加载类。比如JDBC是一个操作数据库的标准，而不同数据库例如mysql需要提供对JDBC标准的实现，也就是driver包，JDBC相关接口在rt.jar中，会由Bootstrap ClassLoader加载，数据库driver则是我们自己放到classpath中，就需要Application ClassLoader或者我们自定义类加载器去加载。Thread的ContextClassLoader默认继承自创建该线程的线程，如果线程没有设置，就使用Application ClassLoader。

## 运行时内存结构
方法区是规范，jdk 8以前用永久代实现，存放被JVM加载的类信息、常量、静态变量等；jdk 8开始，用元空间（meta space）实现方法区，存放了类信息、运行时常量池等，而常量池和静态变量在堆中存放。


## Class的文件结构
类字节码中的常量池包含了链接表（对其他class文件的引用）

Java语言规范：https://docs.oracle.com/javase/specs/jls/se11/html/index.html

Java虚拟机规范：https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-4.html#jvms-4.1

class文件格式使用一个类似C语言的伪结构表示如下：
```
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```
> u1、u2、u4 分别表示无符号的一字节、两字节或四字节数量。

* magic（魔数）：唯一作用是确定这个文件是否为一个能被虚拟机接收的 Class 文件，值为0xCAFEBABE
* minor_version（次要版本）,major_version（主要版本）
* constant_pool_count：常量池数量，constant_pool：常量池数组。常量池中包含引用的各种字符串常量、类和接口名称、字段名称和其他被ClassFile结构引用的常量（比如方法名）。常量池从index为1开始使用。
* access_flags：表示访问权限，例如`ACC_PUBLIC`为public、`ACC_FINAL`为final，`ACC_SUPER`表示会使用`invokespecial`指令调用超类方法。还有其他一些枚举，最终的值是若干个标记的或运算结果。
* this_class、super_class、interfaces_count、interfaces数组：类索引、父类索引和接口索引集合可以确定Java类的继承关系，它们的值为常量池中的index。
* fields_count和fields[]集合：field_info结构表示所有字段，包括类和接口中的静态变量和成员变量，但不包括继承自父类、父接口中的。
```
field_info {
    u2             access_flags; // ACC_PUBLIC、ACC_PRIVATE、ACC_PROTECTED、ACC_STATIC、ACC_VOLATILE 等枚举
    u2             name_index; // 在常量池中的index，可以在常量池中得到字段名称
    u2             descriptor_index; // 指向常量池中的一个CONSTANT_Utf8_info类型数据，比如 I 表示int类型，F 表示float，
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```
descriptor_index指向的常量池数据具体含义：

|FieldType    |type	      |Type	|Interpretation|
|---          |---        |---|---|
|B	          |byte	      |signed byte|
|C	          |char	      |char类型|
|D	          |double	  |double类型|
|F	          |float	  |float类型|
|I	          |int	      |int类型|
|J	          |long	      |long类型|
|L ClassName; |reference  |ClassName 对应类的实例，比如Object类型就是 Ljava/lang/Object|
|S	          |short	  |short类型|
|Z	          |boolean	  |boolean类型|
|[	          |reference  |一个数组维度，比如 double[][] 类型就是 [[D |

* methods_count、methods[]集合：methods[]中数据类型为method_info结构，表示类或接口中的成员方法、静态方法、构造方法、静态代码块，不包含继承的方法。
```
method_info {
    u2             access_flags;  // 描述符，比如public等访问权限、static、final、native、abstract等
    u2             name_index;  // 方法名称索引
    u2             descriptor_index;  // 方法描述索引
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```
属性有很多，比如 Code_attribute（代码）、Exceptions_attribute（异常信息）、InnerClasses_attribute（内部类）、EnclosingMethod_attribute、Synthetic_attribute、Signature_attribute、StackMapTable_attribute等

Code属性
包含了方法内的指令和辅助信息，
Code_attribute {
    u2 attribute_name_index; // 属性名称在常量池中的index
    u4 attribute_length; // 属性的字节长度
    u2 max_stack; // 操作数栈的最大深度（和max_locals一样，用于jvm分配空间）
    u2 max_locals;  // 局部变量最大数量
    u4 code_length; // code集合的长度
    u1 code[code_length]; // jvm指令的实际字节数组
    u2 exception_table_length; 异常表长度
    {   u2 start_pc; // try-catch的开始指令在code集合中的位置
        u2 end_pc; // try-catch的结束指令在code集合中的位置
        u2 handler_pc; // 异常处理的开始指令在code集合中的位置
        u2 catch_type; // 常量池中的index，对应捕获的异常类型，
    } exception_table[exception_table_length]; // 异常表每一项都表示一个异常处理try-catch
    u2 attributes_count; // Code属性内的属性数量
    attribute_info attributes[attributes_count]; Code属性内的属性集合
}
Code中又嵌套了其他属性，比如LineNumberTable、LocalVariableTable、LocalVariableTypeTable、StackMapTable

LineNumberTable
表示源码和字节码指令的行数对应，比如异常堆栈和debug都会用到。
LineNumberTable_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 line_number_table_length;
    {   u2 start_pc; // 指令index
        u2 line_number; // 对应源码行数
    } line_number_table[line_number_table_length];
}

LocalVariableTable
描述局部变量和源码中变量的信息
LocalVariableTable_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 local_variable_table_length;
    {   u2 start_pc; // [start_pc, start_pc + length)为此局部变量可见的指令区间
        u2 length;
        u2 name_index; // 常量池中的index，对应变量名
        u2 descriptor_index; // 常量池中的index，对应变量类型
        u2 index; // 在本地变量表中的index
    } local_variable_table[local_variable_table_length];
}

MethodParameters 
记录方法的参数信息，Java8新增，需要编译时加上-parameters参数，用于支持反射获取方法参数名称。
MethodParameters_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u1 parameters_count;
    {   u2 name_index;
        u2 access_flags;
    } parameters[parameters_count];
}

StackMapTable
在Java 1.6添加，用于JVM加载class时的校验流程执行类型检查验证，这个是字节码里面最复杂的结构。官方文档中的验证流程介绍：https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-4.html#jvms-4.10，这个验证的介绍篇幅很多。

对于JVM来说，一个字节码文件的数据可能是异常的，比如你可以随便手写一个字节码文件。所以Java虚拟机在加载类文件时会执行验证，检查类文件数据是否合法。其中有一项就是检查方法的字节码指令是否正确排序、是否对正确类型的值进行操作等等。JVM引入了StackMapTable进行辅助，编译生成字节码文件时，在Code属性中使用StackMapTable来记录指令执行前后局部变量和操作数栈的值类型和数量，用于验证流程使用。

简单来说，比如字节码中并没有显式的值类型，在验证阶段就需要逐步分析每条指令推断实时的局部变量和操作数栈信息，对于顺序的代码逐步计算验证即可，但如果遇到分支跳转代码，执行的代码流可能性很多，需要穷举所有执行的情况，如果在编译时把每个basic block开始时局部变量和操作数栈情况记录下来，验证此分支时就不需要从分支之前穷举推导验证，可以节省很多时间，也就是把时间耗在编译时，而不是加载验证的时候。当然，StackMapTable只是减少了部分的验证工作

来自《深入理解Java虚拟机》中的一段话：
这项属性描述了方法体中所有的基本块 （Basic Block，按照控制流拆分的代码块）开始时本地变量表和操作栈应有的状态，在字节码验证期间，就不需要根据程序推导这些状态的合法性，只需要检查StackMapTable属性中 的记录是否合法即可。这样将字节码验证的类型推导转变为类型检查从而节省一些时间。 理论上StackMapTable属性也存在错误或被篡改的可能，所以是否有可能在恶意篡改了 Code属性的同时，也生成相应的StackMapTable属性来骗过虚拟机的类型校验则是虚拟机设 计者值得思考的问题。

但如果每一行都加入栈帧映射（stack map frame）记录局部变量和操作数的值类型会很占空间，所以JVM做了优化，每个分支跳转之后使用一个stack map frame用于表示栈帧变化，比如if-else，try-catch等。而直落分支会省略记录，因为直落分支并不会产生 frame 的变化。分支内仍然使用旧的类型推导方式。

那么如下代码，看起有三段代码块（JVM文档中称为basic block）：
// Java代码  
public class Foo {  
    public void foo() {  
        // basic block 1 start  
        int i = 0;  
        int j = 0;  
        // basic block 1 end  
        if (i > 0) {
          // basic block 2 start  
          int k = 0;  
          // basic block 2 end  
        }  
        // basic block 3 start  
        int l = 0;  
        // basic block 3 end  
    }  
}  

根据前面说的优化规则，实际只有两个block：
public class Foo {  
    public void foo() {  
        // basic block 1 start  
        int i = 0;  
        int j = 0;  
        if (i > 0) {  
          int k = 0;  
          // basic block 1 end  
        }  
        // basic block 2 start  
        int l = 0;  
        // basic block 2 end  
    }  
}  
而这个方法就会有一个StackMapTable属性表，里面只有一个stack frame map记录。（本来应该是两个记录，但第一个是隐式的，没有记录在属性表里，应该也是为了节省空间）

使用javap命令查看这个foo方法的字节码：
public void foo();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=4, args_size=1
         0: iconst_0
         1: istore_1
         2: iconst_0
         3: istore_2
         4: iload_1
         5: ifle          10
         8: iconst_0
         9: istore_3
        10: iconst_0
        11: istore_3
        12: return
      LineNumberTable:
        line 52: 0
        line 53: 2
        line 55: 4
        line 57: 8
        line 61: 10
        line 63: 12
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      13     0  this   Lcom/example/myapplication/Test;
            2      11     1     i   I
            4       9     2     j   I
           12       1     3     l   I
      StackMapTable: number_of_entries = 1
        frame_type = 253 /* append */
          offset_delta = 10
          locals = [ int, int ]
隐式的第一个stack map frame是根据方法签名计算出来的，这个例子foo是个成员方法，没有声明参数，所以参数自只有一个隐式的this。所以隐式的第一个stack map frame操作数栈为空，局部变量为[this]。

下一个基本块从字节码偏移量10开始。此处变量k已经不在作用域，所以局部变量的内容应该是：[ this, int, int ]，
此时比上一个基本块开头处的状态多了2个局部变量，类型分别是[ int, int ]，所以就有了上面StackMapTable中的append类型stack map frame。

StackMapTable 内有各种类型的属性，比如上面才看到的append类型是为了不用记录全部数据，而只记录相对上一个stack map frame增加的数据，用于减少空间占用，下面将依次介绍。

StackMapTable属性的结构：
StackMapTable_attribute {
    u2              attribute_name_index;
    u4              attribute_length;
    u2              number_of_entries;
    stack_map_frame entries[number_of_entries]; // 每个entry描述一个stack map frame
}
// stack_map_frame的多种类型
union stack_map_frame {
    same_frame;
    same_locals_1_stack_item_frame;
    same_locals_1_stack_item_frame_extended;
    chop_frame;
    same_frame_extended;
    append_frame;
    full_frame;
}

full_frame：记录完整 locals 和 stack 中的值类型， offset_delta 用来计算当前stack map frame对应的指令位置，也就是对应的basic block的第一个指令的index。如果前一个frame是第一个隐式frame，则对应的指令位置就是offset_delta，如果前一个frame不是第一个，则对应的指令位置为上一个 frame 对应的index + offset_delta + 1。
full_frame {
    u1 frame_type = FULL_FRAME; /* 255 */
    u2 offset_delta; // 相对于前一个stack map frame的偏移量
    u2 number_of_locals;
    verification_type_info locals[number_of_locals]; // 记录局部变量中的值类型
    u2 number_of_stack_items;
    verification_type_info stack[number_of_stack_items]; // 记录操作数栈中的值类型
}
其他类型的frame_type都是为了简化，减少字节码数量

same_frame：frame_type 为 0-63 代表当前 frame 的类型是 same_frame，offset_delta等值等于frame_type。
意思是 locals 和上一个 frame 相同，但是 stack 为空
same_frame {
    u1 frame_type = SAME; /* 0-63 */
}

// offset较大时，就使用 same_frame_extended
same_frame_extended {
    u1 frame_type = SAME_FRAME_EXTENDED; /* 251 */
    u2 offset_delta;
}

same_locals_1_stack_item_frame：locals和前一个frame相同。有一个深度为1的操作数stack，offset_delta为frame_type-64
same_locals_1_stack_item_frame {
    u1 frame_type = SAME_LOCALS_1_STACK_ITEM; /* 64-127 */
    verification_type_info stack[1];
}

same_locals_1_stack_item_frame_extended {
    u1 frame_type = SAME_LOCALS_1_STACK_ITEM_EXTENDED; /* 247 */
    u2 offset_delta;
    verification_type_info stack[1];
}

这两个 frame 表示的 stack 都是 empty
chop_frame 表示相比前一个frame，少了最后的 251 - frame_type 个 局部变量，
append_frame 表示  除了增加的额外的frame_type - 251 个局部变量，其他的局部变量和前一个frame一样
chop_frame {
    u1 frame_type = CHOP; /* 248-250 */
    u2 offset_delta;
}

append_frame {
    u1 frame_type = APPEND; /* 252-254 */
    u2 offset_delta;
    verification_type_info locals[frame_type - 251];
}
这两个类型也是为了节省空间，但在添加或删除超过 3 个 locals 的时候，还是会降级到使用 full_frame
有点像视频编码的i、p、b帧的思想

- attributes_count和attributes集合：用于描述class文件的属性，比如SourceFile属性用于描述编译为此字节码文件的源码文件的名称，更完善的信息可以查看文档：https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-4.html#jvms-4.7-320


> jvm指令集大全：https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-7.html

## GC
Java才用可达性算法（根搜索）判断对象是否需要回收。内存回收有标记-清除、复制、标记-整理等算法，因为不同对象有不同的生命周期，所以JVM将内存划为新生代和老生代，新生代采用复制算法，因为新生代每次回收后存活的对象是少数，所以复制并不会付出很多的代价，而老生代使用标记-整理之类的算法。

## 对象的内存布局
使用JOL库可以查看对象的内存布局

普通对象的内存布局：
1. 对象头：mark word（8个字节）、类型指针（在64位虚拟机上，是64位也就8个字节，但可能开启了指针压缩就是4个字节）
2. 实例数据
3. padding对齐：必须被8整除，所以要补齐到8的整数倍字节

> 可以用 java -XX:PrintCommandLineFlags -version，可以查看是否有UseCompressesClassPoints（开启类指针压缩）

数组的内存布局：
1. 对象头：mark word、类型指针
2. 数组长度：4个字节
3. 实例数据
4. padding对齐

使用new创建对象时，会先申请内存，再赋默认值，最后执行初始化代码（构造函数）赋初始值。例如双重检验单例就是使用volatile的有序性来避免这个顺序变化。

## 虚引用
强、软、弱三种引用都比较好理解它的使用场景，而虚引用似乎并没有什么用，通过它并不能拿到引用的对象，那么Java设计它的目的是什么呢？实际上使用它是必须配合ReferenceQueue，在对象被回收时就会在引用队列中取到引用，从而监听到对象被回收了。比如管理堆外内存，对 directByteBuffer 增加一个虚引用，当强引用没有的时候，它会马上被垃圾回收，通过referenceQueue就可以监听到，然后释放堆外内存。弱引用也可以用来做对象回收的监听，但是通过弱引用获取到对象赋值给一个变量，就可以对它增加强引用，而虚引用可以避免这个情况。（Reference内的processPendingReferences方法，判断了Cleaner类，directByteBuffer正是使用了Cleaner）