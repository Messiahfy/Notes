https://learnopengl-cn.github.io/

* OpenGL 规范文档：https://registry.khronos.org/OpenGL/specs/es/3.2/es_spec_3.2.pdf
* OpenGL API文档：https://registry.khronos.org/OpenGL-Refpages/gl4/

# 介绍
OpenGL（Open Graphics Library）是一个平台无关的图形编程接口，OpenGL ES 是基于 OpenGL 的子集，主要针对手机、pad之类的嵌入式设备。OpenGL 定义了一套 API，让我们可以控制 `GPU` 来绘制 2D/3D 图形。而 `Shader` 是 OpenGL 的核心，是运行在 GPU 上的小程序。

OpenGL 本身只是图形编程接口规范，具体的实现是由各个 GPU 厂商的驱动程序完成。

> DirectX（Windows）、Metal（Apple）、Vulkan（Android）则是特定平台的图形编程接口

# OpenGL ES 基础
OpenGL ES 是纯粹的 GPU 渲染绘图 API，只负责操作、处理GPU 显存中的数据。它的核心工作只有两件事：
1. 将图形、图像数据渲染到帧缓冲区（FrameBuffer）；
2. 读取帧缓冲区中渲染完成的像素数据。

>【核心误区解答：不自定义FBO也能渲染的原因】很多初学者会疑惑：日常直接渲染到 WindowSurface 屏幕，没有创建FBO，为什么还说渲染目标是帧缓冲区？
> 答案：所有OpenGL渲染，必须渲染到帧缓冲区，没有任何例外。
> 帧缓冲区分为两种，本质是同一种东西：
> - 默认帧缓冲区（FBO=0）：由EGL自动创建、绑定到EGLSurface（WindowSurface/PbufferSurface），开发者无需手动创建。直接渲染到Surface，本质就是渲染到这个系统自带的默认FBO。
> - 自定义帧缓冲区（自定义FBO）：由开发者手动创建，用于离屏渲染、画面预处理，不绑定屏幕。

OpenGL ES 所有的绘制工作，都是由可**编程着色器（Shader）**和**固定功能渲染单元**配合完成，最终对三角形、线段等图元进行渲染计算。

## 状态机
OpenGL ES 是一个大型状态机，所有渲染行为都由当前的「上下文状态」决定。我们可以通过 API 修改渲染状态，例如：开启/关闭深度测试、开启混合、设置纹理采样方式、设置绘制图元类型（三角形/线段）等。状态一旦修改，后续所有的绘制命令都会遵循当前最新状态执行，直到状态被再次修改。

## 可编程渲染管线
OpenGL 的渲染过程是一系列分阶段的并行处理流程，现代管线关键阶段包括：
1. 顶点着色器：处理每个顶点的位置变换（如模型-视图-投影矩阵计算），输出裁剪空间坐标。
2. 图元装配：将顶点按指定模式（如 GL_TRIANGLES）组装成几何图元（三角形、线段等）。
3. 光栅化：将图元转换为屏幕像素的候选片段（Fragment），自动插值顶点属性（如颜色、纹理坐标）。
4. 片段着色器（像素着色器）：计算每个片段的最终颜色（可实现光照、纹理采样等效果）。
5. 测试与混合：通过深度测试（GL_DEPTH_TEST）、模板测试和混合操作（GL_BLEND）决定片段是否写入帧缓冲。

> 开发者重点关注 `顶点着色器` 和 `片段着色器`

## 使用流程
1. EGL 环境初始化
Display → Config → Context → WindowSurface/Pbuffer → eglMakeCurrent 打通 GL 环境

> EGL 会提前为 OpenGL ES 提供一块默认屏幕缓冲区（WindowSurface 默认帧缓冲）

2. 着色器编译链接
  * 写顶点着色器、片元着色器源码
  * glCreateShader→shaderSource→compile→attach→link→glUseProgram (编译链接、绑定启用)

3. CPU → GPU 传入数据
  * 顶点数据（xy坐标 / UV / 颜色）：VBO+VAO+EBO
  * 图片数据：Texture 纹理。 CPU 位图 使用 glTexImage2D 上传普通纹理，或者外部视频/相机提供 EGLImage+OES 外部纹理
  * 全局参数：Uniform 变量（矩阵、颜色、滤镜系数，渲染时动态修改）

> UV 就是纹理坐标，决定这个顶点去【贴图图片】上取哪个位置的像素颜色，U = 横向、V = 纵向，等价图片的 X、Y

> xy 决定画的位置，uv 决定去哪取图，或者说怎么贴图。

4. 渲染循环
执行相关 gl 函数

5. 资源销毁

## GL 核心对象

### 缓冲对象 Buffer Object
所有 BufferObject 都开辟在 GPU 显存中，由 `glGenBuffers`、`glGenFramebuffers` 创建，`glBindBuffer`、`glBindFramebuffer` 绑定使用；分为两大阵营：「存放输入顶点数据」、「存放渲染输出画面」
* 输入类：VBO、VBO、EBO（决定画什么）
* 输出类：FBO（决定画到哪里）

#### 1. VBO 顶点缓冲对象

`Vertex Buffer Object`，保存顶点数据：顶点坐标、颜色、UV。将顶点数据上传至 GPU 显存，避免 CPU-GPU 频繁传输

GL_ARRAY_BUFFER，CPU 数据上传显存，提速。

> 顶点（Vertex）不代表坐标，坐标是顶点的一部分，是顶点的其中一个属性，顶点还可以有其他属性，如颜色，纹理，法线等等。如果一个人告诉你，我有一个顶点，是（0.5，-0.5），这是不完全正确的。

#### 2. EBO (IBO) 索引缓冲区
`Element Buffer Object` 或 `Index Buffer Object`。保存顶点索引，配合 VBO 复用顶点减少数据量。比如矩形 4 个顶点，靠索引数字拼两个三角形，不用重复存 6 个点

GL_ELEMENT_ARRAY_BUFFER

#### 3. VAO 顶点数组对象
`Vertex Array Object`，记录 VBO/EBO 的绑定状态和顶点属性指针的配置，之后可以直接复用，实现一键切换渲染状态。

通俗的说，VAO 的核心机制就是“快照存档”。它不会存储顶点数据本身，而是记录顶点属性的配置状态（如数据格式、绑定关系等），后续通过绑定 VAO 一键恢复这些状态，实现高效复用。

```cpp
GLuint VAO, VBO;
glGenVertexArrays(1, &VAO);
glGenBuffers(1, &VBO);

glBindVertexArray(VAO);  // 激活 VAO 状态记录
glBindBuffer(GL_ARRAY_BUFFER, VBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

// 配置顶点属性（位置）
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);  // 启用属性

glBindVertexArray(0);  // 解绑 VAO（保存配置）


glUseProgram(shaderProgram);
glBindVertexArray(VAO);  // 一键恢复所有顶点配置
glDrawArrays(GL_TRIANGLES, 0, 3);  // 直接绘制
glBindVertexArray(0);     // 解绑（可选）
```
`glVertexAttribPointer`、`glEnableVertexAttribArray`、`EBO 绑定关系` 的相关状态都会被 VAO 保存。之后可以直接复用。


#### 4. FBO 帧缓存对象
FBO（Frame Buffer Object，帧缓冲对象）本质上是一个渲染目标（Render Target）管理器。

它本身并不存储图像数据，而是负责管理：

* 颜色缓冲（Color Attachment）
* 深度缓冲（Depth Attachment）
* 模板缓冲（Stencil Attachment）

这些真正存储数据的附件（Attachment）绑定关系。

可以理解为：
|对象| 类比|
|---|---|
|FBO| 画板|
|Texture| 可重复利用的画布|
|RenderBuffer（RBO）| 专用画布|
|glDrawArrays| 在画板上作画|

默认情况下，OpenGL 会直接渲染到屏幕，但有些场景需要在渲染前对图像做处理，例如滤镜。中间结果不能立即显示，需要先渲染到纹理。这个方式就叫离屏渲染。

默认 FrameBuffer 由 EGL / 窗口系统创建。
```
// 表示绑定默认的FrameBuffer
glBindFramebuffer(GL_FRAMEBUFFER, 0);
```
用户也可以使用 `glGenFrameBuffers` 创建自定义的 FBO。

FBO自身不存储数据，它真正存储数据的是附件：
```
FBO
 ├─ Color Attachment
 ├─ Depth Attachment
 └─ Stencil Attachment
```

使用`glFramebufferTexture2D(...)`就就可以把纹理挂载到FBO上。

FBO还可以挂载 RBO（渲染缓冲），RBO可以使用 `glFramebufferRenderbuffer` 创建。RBO仅用于“GPU内部一次性消耗”的数据（如深度/模板测试、MSAA中间结果），当且仅当数据无需被着色器采样或CPU读取时使用。其他场景（尤其是颜色输出）应优先选择纹理附件。


### Texture 纹理对象
Texture 是 GPU 中实际存储像素数据的资源（如 RGBA/YUV 值），其内容可通过着色器动态采样（例如 texture(sampler2D, uv)）。它作为可采样的图像数据载体，既可用于普通贴图，也可作为 FBO 的附件接收动态渲染结果。

## 坐标系
OpenGL 世界坐标，是范围是 -1 ～ 1 的归一化坐标，原点在中心。

OpenGL 纹理坐标，其实就是屏幕坐标，是2D归一化坐标，定义纹理图像如何映射到物体表面，标准的纹理坐标原点是在屏幕的左下方，而Android系统坐标系的原点是在左上方。

像素点应该显示在哪个位置由世界坐标决定，颜色所在的位置由纹理坐标决定

## Shader 着色器
着色器是在 GPU 中运行的程序

* 顶点着色器 VertexShader：顶点坐标变换（3D→屏幕坐标）、输出 UV
* 片元着色器 FragmentShader：像素配色、纹理采样、滤镜计算
编译流程：创建 shader→源码附着→编译→创建 program→链接 shader→glUseProgram 启用。

> ⚠️ 片元/像素shader，可以认为 fragment 就是像素，每个像素都会执行一次片元着色器的代码，为每个像素确定颜色。比如绘制三角形，顶点着色器会调用3次，每个顶点一次；像素着色器则是根据三角形的大小，范围内有多少个像素就执行多少次 

### OpenGL 着色器语言 GLSL
在OpenGL 2.0以后，加入了新的可编程渲染管线，可以更加灵活的控制渲染，使用GLSL语言编写。着色器需要编译、然后链接到OpenGL程序中。

`gl_` 作为GLSL保留前缀只能用于内部变量。还有一些GLSL内置函数名称是不能够作为变量的名称。

#### 1. 数据类型

|类型|描述|
|---|---|
|int、float、double、bool|和常规语言一样|
|vec2、vec3、vec4|2、3、4 维浮点数向量|
|bvec2、bvec3、bvec4|2、3、4 维布尔向量|
|ivec2、ivec3、ivec4|2、3、4 维整型向量|
|mat2、mat3、mat4|2x2、3x3、4x4 浮点矩阵|
|mat2x2、mat2x3、mat2x4|2x2、2x3、2x4 浮点矩阵|
|mat3x2、mat3x3、mat3x4|3x2、3x3、3x4 浮点矩阵|
|mat4x2、mat4x3、mat4x4|4x2、4x3、4x4 浮点矩阵|
|sampler1D、sampler2D、sampler2D|1、2、3 维纹理|
|samplerCube|cube map纹理|
|sampler1DShadow、sampler2DShadow|1、2 维深度纹理|

#### 2. 结构体
可以使用C/C++中的结构体
```
struct light {
 float intensity;
 vec3 position;
} lightVar;
```

#### 3. 数组
可以使用一维数组

#### 4. 存储修饰符
|修饰符|描述|
|---|---|
|默认|局部变量|
|const|声明变量或函数参数为常量，只读|
|attribute|全局变量，仅用于顶点着色器，在顶点着色器内是只读的，由用户代码调用OpenGL API赋值|
|uniform|全局变量，可用于顶点和片段着色器，可在顶点和片段着色器中共享访问（声明一样的话）|
|varying|顶点着色器可以读写varying，片段着色器只能读取，可以在顶点和片段着色器传递数据，在片段着色器中得到的varying经过了光栅化的插值|
|centroid varying|多重采样情况使用|
|invariant|不变限定符，用于消除着色器编译优化带来的计算误差，强制保证特定的输出变量在不同的着色器程序或不同的渲染通道中，计算结果绝对精确一致。因为编译器会对着色器代码进行各种内部优化，导致一个被称为方差（Variance）的问题，这种微小的误差在单通道渲染中通常不可见，但在多通道算法（Multi-pass）中会导致严重的视觉伪影，例如深度冲突（Z-fighting）或边缘闪烁|

**说明：**
* 局部变量要么不用修饰符，要么只能使用const修饰
* `attribute`变量通过OpenGL vertex API或作为顶点数组的一部分传入。仅用于float、float 向量和矩阵类型变量。attribute变量都会有一个内置的变量名，用于用户程序访问。
* `uniform`变量在着色器代码初始化，或者通过OpenGL API初始化。
* 只有`attribute`和`uniform`声明的变量可以在用户代码中访问和赋值。所以例如片段着色器需要用户代码设置颜色，就需要使用`uniform`，因为`attribute`只能在顶点着色器中使用。

#### 5. 参数修饰符
OpenGL 3版本

|修饰符|描述|
|---|---|
|默认|和in一样|
|in|只读、数据流入当前着色器|
|out|只写、数据流出当前着色器|
|inout|可读可写、数据先进来，修改后再传出去|

比如在着色器中使用 `in` 和 `out`：
```glsl
// 顶点着色器
layout(location=0) in vec3 aPos;
out vec3 v_Color; //向外输出颜色

void main(){
    // aPos.x = 1.0; ❌ 报错，in变量禁止写入

    gl_Position = vec4(aPos,1.0);
    //根据坐标生成RGB，输出给片元
    v_Color = aPos + vec3(0.5);
}
```

```glsl
// 片元着色器
in vec3 v_Color; // 从顶点着色器接收插值后的颜色
out vec4 fragColor;

void main(){
    fragColor = vec4(v_Color,1.0);
}
```

`inout` 则是既能读原值，又能修改后输出。

#### 6. 预处理
支持C/C++里面的部分预处理指令
```glsl
#define
#undef

#if
#ifdef
// 省略其他。。。
```

GLSL 特有的预处理指令：
```glsl
#version // 指定当前 GLSL 的版本号

#extension // 用于启用、禁用或要求特定的 OpenGL 扩展功能

// 省略其他
```

#### 7. 操作符和表达式

普通类型构造函数
```glsl
int(bool) // converts a Boolean value to an int
int(float) // float值转换为int，会丢弃小数部分
float(bool) // converts a Boolean value to a float
float(int) // converts an integer value to a float
bool(float) // converts a float value to a Boolean
bool(int) // 0为false，非0为true


vec3(float) // initializes each component of with the float
vec4(ivec4) // makes a vec4 with component-wise conversion
vec2(float, float) // initializes a vec2 with 2 floats
ivec3(int, int, int) // initializes an ivec3 with 3 ints
bvec4(int, int, float, float) // uses 4 Boolean conversions
vec2(vec3) // drops the third component of a vec3
vec3(vec4) // drops the fourth component of a vec4
vec3(vec2, float) // vec3.x = vec2.x, vec3.y = vec2.y, vec3.z = float
vec3(float, vec2) // vec3.x = float, vec3.y = vec2.x, vec3.z = vec2.y
vec4(vec3, float)
vec4(float, vec3)
vec4(vec2, vec2)


mat2(vec2, vec2); // one column per argument
mat3(vec3, vec3, vec3); // one column per argument
mat4(vec4, vec4, vec4, vec4); // one column per argument

mat2(float, float, // first column
 float, float); // second column
```

数组构造
```
float c[3] = float[3](5.0, 7.2, 1.1);
float d[3] = float[](5.0, 7.2, 1.1);
```

向量、矩阵类型支持运算符重载操作
```
vec3 v, u;
float f;
v = u + f;

// 相当于
v.x = u.x + f;
v.y = u.y + f;
v.z = u.z + f;
```

#### 8. 内置变量
顶点着色器内置变量：
* `gl_Position(vec4类型)`：顶点着色器中用于输入顶点数据，将用于顶点着色器之后的固定流程
* `gl_PointSize(float类型)`：顶点着色器中用于设置单个点的像素大小，一般默认为1

#### 9. 内置函数

##### 一、基础数值运算函数（最常用：数值截断、插值、阶跃）
* abs(x)：绝对值
* sign(x)：符号，正数 1、负数 - 1、0 为 0
* floor(x)/ceil(x)：向下取整 / 向上取整
* fract(x)：取小数部分 fract(2.3)=0.3
* mod(a,b)：取模、求余数
* clamp(x,min,max)：数值限制在 [min,max]，超边界直接锁在边界
* mix(a,b,t)：线性插值 t∈[0,1]，t=0 取 a、t=1 取 b
* step(edge,x)：阶跃函数，x<edge→0，否则 1（黑白分界）
* smoothstep(min,max,x)：平滑过渡，0~1 平滑渐变，做羽化、软边缘

#### 二、指数、幂、开方类
1. pow(base,exp)：幂运算
2. exp(x)：自然指数；exp2(x)
3. log(x)：自然对数；log2(x)：以 2 为底对数
4. sqrt(x)：平方根；inversesqrt(x)
​	
 
#### 三、三角函数（参数全部是弧度）
1. 基础正余弦sin/cos/tan 三角函数
2. 反三角函数asin/acos/atan(y,x)，atan 双参数求象限夹角
3. 弧度转换radians(角度)：角度→弧度；degrees(弧度)：弧度→角度

#### 四、几何函数
* dot(a,b)：点积，返回 float，算夹角、漫反射亮度
* cross(a,b)：叉积 (仅 vec3)，生成垂直新法线
* normalize(v)：向量归一化，长度变成 1
* length(v)：向量模长
* distance(a,b)：两点距离 = length (a-b)
* reflect(I,N)：反射向量（镜面高光）
* refract(I,N,eta)：折射向量（玻璃、水）
* faceforward(N,I,Nref)：反向修正法线方向

#### 五、矩阵专用函数（300es 支持）
* matrixCompMult(m1,m2)：矩阵对应元素相乘（≠数学矩阵乘法）
* outerProduct(vecA,vecB)：向量外积生成矩阵
* transpose(mat)：矩阵转置
* determinant(mat)：求行列式
* inverse(mat)：矩阵求逆

#### 六、Texture 采样函数（仅片元着色器可用）
* texture(sampler,uv)：普通 2D 纹理采样
* textureLod(sampler,uv,lod)：手动指定 mipmap 层级
* 外部 OES 纹理：texture(samplerExternalOES,uv)（相机 / 视频专用）


# 实际 API 使用流程（Android为例画三角形）

## 直接使用 CPU 内存数据
```java
public class TriangleRender implements GLSurfaceView.Renderer {
    //1.顶点着色器源码：300 es，固定location=0输入顶点
    // #version 可选配置
    // precision mediump float; 设置精度中等，也是可选配置
    private final String VERTEX_SOURCE = "#version 300 es\n" +
            "precision mediump float;\n" +
            "layout(location = 0) in vec3 aPos;\n" +
            "void main(){\n" +
            "    gl_Position = vec4(aPos, 1.0);\n" +
            "}";

    //2.片元着色器：固定输出红色
    private final String FRAG_SOURCE = "#version 300 es\n" +
            "precision mediump float;\n" +
            "out vec4 fragColor;\n" +
            "void main(){\n" +
            "    fragColor = vec4(1.0f, 0.0f, 0.0f, 1.0f);\n" +
            "}";

    //三角形NDC坐标：[-1~1] 左下、右下、顶部中点
    private final float[] TRI_DATA = {
            -0.5f, -0.5f, 0.0f,
             0.5f, -0.5f, 0.0f,
             0.0f,  0.5f, 0.0f
    };

    private int glProgram;          //着色器程序ID
    private FloatBuffer vertexBuf;  //CPU内存顶点缓冲区（无VBO，数据存在RAM）

    /**
     * Surface创建时执行一次：编译链接shader、初始化顶点Buffer
     */
    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        //步骤1：编译生成Program程序
        glProgram = createProgram(VERTEX_SOURCE, FRAG_SOURCE);

        //步骤2：把float数组转为OpenGL可用的Native内存FloatBuffer
        //OpenGL不能直接读取Java堆内存，必须开辟操作系统原生内存
        vertexBuf = FloatBuffer.allocate(TRI_DATA.length);
        //设置字节序为当前CPU原生字节序（安卓ARM小端）
        vertexBuf.order(ByteOrder.nativeOrder());
        //把浮点数组数据写入Buffer
        vertexBuf.put(TRI_DATA);
        //指针重置到起始位置，后续读取从头开始
        vertexBuf.position(0);
    }

    /**
     * 窗口尺寸变化时调用：设置视口大小
     */
    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        //视口：渲染图像铺满整个窗口
        GLES30.glViewport(0, 0, width, height);
    }

    /**
     * 每一帧绘制执行：【无VBO、无VAO，每帧配置顶点属性】
     */
    @Override
    public void onDrawFrame(GL10 gl) {
        //1.清空屏幕为黑色
        GLES30.glClearColor(0f, 0f, 0f, 1f);
        GLES30.glClear(GLES30.GL_COLOR_BUFFER_BIT);

        //2.启用着色器程序
        GLES30.glUseProgram(glProgram);

        //====================关键：不绑定任何VBO，使用CPU内存数据====================
        //参数说明：
        //参数1：属性下标location=0 对应shader layout(location=0) in aPos
        //参数2：单个顶点分量数量 3个(x,y,z)
        //参数3：数据类型 float
        //参数4：是否归一化 false（坐标不需要映射0~1）
        //参数5：步长：一个顶点占用字节 3*4(float=4字节)
        //参数6：数据源：没有VBO，直接填CPU的FloatBuffer地址
        GLES30.glVertexAttribPointer(0,3, GLES30.GL_FLOAT,false,3*4,vertexBuf);

        //启用0号顶点属性通道
        GLES30.glEnableVertexAttribArray(0);

        //3.绘制三角形：从第0个顶点开始，一共3个顶点
        GLES30.glDrawArrays(GLES30.GL_TRIANGLES,0,3);

        //关闭属性通道（可选，规范写法）
        GLES30.glDisableVertexAttribArray(0);
    }

    //工具函数：编译单个着色器
    private int compileShader(int type, String code){
        int shaderId = GLES30.glCreateShader(type);
        GLES30.glShaderSource(shaderId,code);
        GLES30.glCompileShader(shaderId);
        return shaderId;
    }

    //工具函数：链接顶点+片元shader生成Program
    private int createProgram(String vertCode,String fragCode){
        int vert = compileShader(GLES30.GL_VERTEX_SHADER,vertCode);
        int frag = compileShader(GLES30.GL_FRAGMENT_SHADER,fragCode);
        int program = GLES30.glCreateProgram();
        GLES30.glAttachShader(program,vert);
        GLES30.glAttachShader(program,frag);
        GLES30.glLinkProgram(program);
        //编译链接完成删除无用shader
        GLES30.glDeleteShader(vert);
        GLES30.glDeleteShader(frag);
        return program;
    }
}
```
* layout(location = 0)：指定变量在显存中布局位置，相关代码中就可以直接使用 0，否则需要用 `glGetAttribLocation` 查找属性的位置后再使用

## 使用 VBO、VAO
使用 vbo，可以把顶点数据一次性存入GPU 显存，后续绘制不再从 CPU 内存取数据，性能更好

> 可以只用VBO，不用VAO，但使用VAO性能更好

```java
public class TriangleRender implements GLSurfaceView.Renderer {
    //顶点着色器
    private final String vertSource = "#version 300 es\n" +
            "precision mediump float;\n" +
            "layout(location = 0) in vec3 aPos;\n" +
            "void main(){\n" +
            "    gl_Position = vec4(aPos, 1.0);\n" +
            "}";
    //片元着色器：红色
    private final String fragSource = "#version 300 es\n" +
            "precision mediump float;\n" +
            "out vec4 fragColor;\n" +
            "void main(){\n" +
            "    fragColor = vec4(1.0f,0.0f,0.0f,1.0f);\n" +
            "}";

    //NDC三角形三点：左下、右下、中上
    private final float[] vertexData = {
            -0.5f, -0.5f, 0.0f,
             0.5f, -0.5f, 0.0f,
             0.0f,  0.5f, 0.0f
    };

    private int program;
    private int vaoId;
    private int vboId;

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        //1.编译链接着色器
        program = createProgram(vertSource, fragSource);

        //2.把数组转为Native缓冲，仅用来一次性上传VBO
        FloatBuffer buffer = FloatBuffer.allocate(vertexData.length);
        buffer.order(ByteOrder.nativeOrder());
        buffer.put(vertexData);
        buffer.position(0);

        //=====VAO+VBO初始化（只执行1次）=====
        int[] vaoArr = new int[1];
        int[] vboArr = new int[1];
        //创建VAO对象
        GLES30.glGenVertexArrays(1, vaoArr, 0);
        vaoId = vaoArr[0];
        //创建VBO对象
        GLES30.glGenBuffers(1, vboArr, 0);
        vboId = vboArr[0];

        //【关键】绑定VAO：之后所有VBO、属性配置都会记录进这个VAO
        GLES30.glBindVertexArray(vaoId);

        //绑定VBO，把顶点数据送入GPU显存
        GLES30.glBindBuffer(GLES30.GL_ARRAY_BUFFER, vboId);
        GLES30.glBufferData(GLES30.GL_ARRAY_BUFFER, vertexData.length*4, buffer, GLES30.GL_STATIC_DRAW);

        //配置顶点属性：配置自动存入当前绑定的VAO
        //参数：属性序号0，3个float，浮点，不归一，步长12字节，VBO内部偏移0
        GLES30.glVertexAttribPointer(0,3, GLES30.GL_FLOAT, false,3*4,0);
        //启用0号属性
        GLES30.glEnableVertexAttribArray(0);

        //解绑VBO
        GLES30.glBindBuffer(GLES30.GL_ARRAY_BUFFER,0);
        //解绑VAO（配置保存完毕）
        GLES30.glBindVertexArray(0);
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GLES30.glViewport(0,0,width,height);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        //清屏黑色
        GLES30.glClearColor(0,0,0,1);
        GLES30.glClear(GLES30.GL_COLOR_BUFFER_BIT);

        //启用着色器程序
        GLES30.glUseProgram(program);

        //【渲染极简】只需要绑定VAO，自动恢复VBO+所有属性配置
        GLES30.glBindVertexArray(vaoId);
        //直接绘制
        GLES30.glDrawArrays(GLES30.GL_TRIANGLES,0,3);

        GLES30.glBindVertexArray(0);
    }

    //编译单个着色器
    private int compileShader(int type, String src){
        int id = GLES30.glCreateShader(type);
        GLES30.glShaderSource(id,src);
        GLES30.glCompileShader(id);
        return id;
    }

    //链接生成program
    private int createProgram(String vert, String frag){
        int vShader = compileShader(GLES30.GL_VERTEX_SHADER,vert);
        int fShader = compileShader(GLES30.GL_FRAGMENT_SHADER,frag);
        int pro = GLES30.glCreateProgram();
        GLES30.glAttachShader(pro, vShader);
        GLES30.glAttachShader(pro, fShader);
        GLES30.glLinkProgram(pro);
        GLES30.glDeleteShader(vShader);
        GLES30.glDeleteShader(fShader);
        return pro;
    }
}
```

## 使用纹理 texture
```java
public class TextureRender implements GLSurfaceView.Renderer {
    private final Context mContext;

    //===== Shader源码 =====
    private final String vertSrc = "#version 300 es\n" +
            "precision mediump float;\n" +
            "layout(location=0) in vec2 aPos;\n" +    //顶点坐标
            "layout(location=1) in vec2 aUV;\n" +     //纹理坐标
            "out vec2 vUV;\n" +                      //传给片元UV
            "void main(){\n" +
            "    gl_Position = vec4(aPos,0.0,1.0);\n" +
            "    vUV = aUV;\n" +
            "}";

    private final String fragSrc = "#version 300 es\n" +
            "precision mediump float;\n" +
            "in vec2 vUV;\n" +
            "uniform sampler2D u_Texture;\n" +       //纹理采样器
            "out vec4 fragColor;\n" +
            "void main(){\n" +
            "    fragColor = texture(u_Texture, vUV);\n" + //纹理采样取色
            "}";

    //矩形数据：[x,y,u,v] 四个顶点 NDC
    private final float[] vertexArr = {
            -0.5f,  0.5f, 0f,1f, //左上
            -0.5f, -0.5f, 0f,0f, //左下
             0.5f, -0.5f, 1f,0f, //右下
             0.5f,  0.5f, 1f,1f  //右上
    };
    //索引：两个三角形拼成矩形
    private final int[] indexArr = {0,1,2, 0,2,3};

    private int program;
    private int vaoId,vboId,eboId;
    private int texId; //纹理ID
    private int texLoc;//shader中sampler2D的位置

    public TextureRender(Context ctx){
        mContext = ctx;
    }

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        //1.编译着色器
        program = createProgram(vertSrc,fragSrc);
        //获取uniform sampler2D的下标
        texLoc = GLES30.glGetUniformLocation(program,"u_Texture");

        //2.初始化顶点Buffer
        FloatBuffer vBuf = FloatBuffer.allocate(vertexArr.length);
        vBuf.order(ByteOrder.nativeOrder());
        vBuf.put(vertexArr);vBuf.position(0);

        IntBuffer iBuf = IntBuffer.allocate(indexArr.length);
        iBuf.order(ByteOrder.nativeOrder());
        iBuf.put(indexArr);iBuf.position(0);

        //3.VAO+VBO+EBO初始化
        int[] vao = new int[1];
        int[] vbo = new int[1];
        int[] ebo = new int[1];
        GLES30.glGenVertexArrays(1,vao,0);
        GLES30.glGenBuffers(1,vbo,0);
        GLES30.glGenBuffers(1,ebo,0);
        vaoId=vao[0];vboId=vbo[0];eboId=ebo[0];

        GLES30.glBindVertexArray(vaoId);
        //绑定VBO存顶点
        GLES30.glBindBuffer(GLES30.GL_ARRAY_BUFFER,vboId);
        GLES30.glBufferData(GLES30.GL_ARRAY_BUFFER,vertexArr.length*4,vBuf,GLES30.GL_STATIC_DRAW);
        //绑定EBO存索引
        GLES30.glBindBuffer(GLES30.GL_ELEMENT_ARRAY_BUFFER,eboId);
        GLES30.glBufferData(GLES30.GL_ELEMENT_ARRAY_BUFFER,indexArr.length*4,iBuf,GLES30.GL_STATIC_DRAW);

        //属性0：坐标vec2，步长4*4=16字节，偏移0
        GLES30.glVertexAttribPointer(0,2,GLES30.GL_FLOAT,false,16,0);
        GLES30.glEnableVertexAttribArray(0);
        //属性1：UV vec2，偏移2*4=8
        GLES30.glVertexAttribPointer(1,2,GLES30.GL_FLOAT,false,16,8);
        GLES30.glEnableVertexAttribArray(1);

        GLES30.glBindVertexArray(0);

        //4.加载图片生成纹理
        texId = loadTexture(R.drawable.test);
    }

    //加载资源图片生成OpenGL纹理
    private int loadTexture(int resId){
        int[] tex = new int[1];
        GLES30.glGenTextures(1,tex,0);
        GLES30.glBindTexture(GLES30.GL_TEXTURE_2D,tex[0]);

        //纹理环绕：超出UV[0,1]重复平铺
        GLES30.glTexParameteri(GLES30.GL_TEXTURE_2D,GLES30.GL_TEXTURE_WRAP_S,GLES30.GL_REPEAT);
        GLES30.glTexParameteri(GLES30.GL_TEXTURE_2D,GLES30.GL_TEXTURE_WRAP_T,GLES30.GL_REPEAT);
        //纹理过滤：缩放线性平滑
        GLES30.glTexParameteri(GLES30.GL_TEXTURE_2D,GLES30.GL_TEXTURE_MIN_FILTER,GLES30.GL_LINEAR);
        GLES30.glTexParameteri(GLES30.GL_TEXTURE_2D,GLES30.GL_TEXTURE_MAG_FILTER,GLES30.GL_LINEAR);

        //解码图片
        Bitmap bmp = BitmapFactory.decodeResource(mContext.getResources(),resId);
        GLUtils.texImage2D(GLES30.GL_TEXTURE_2D,0,GLES30.GL_RGBA,bmp,GLES30.GL_RGBA,GLES30.GL_UNSIGNED_BYTE,0);
        bmp.recycle();
        GLES30.glBindTexture(GLES30.GL_TEXTURE_2D,0);
        return tex[0];
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GLES30.glViewport(0,0,width,height);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        GLES30.glClearColor(0,0,0,1);
        GLES30.glClear(GLES30.GL_COLOR_BUFFER_BIT);

        GLES30.glUseProgram(program);

        //====纹理绑定到0号纹理单元====
        GLES30.glActiveTexture(GLES30.GL_TEXTURE0);
        GLES30.glBindTexture(GLES30.GL_TEXTURE_2D,texId);
        //把采样器u_Texture绑定到TEXTURE0(编号0)
        GLES30.glUniform1i(texLoc,0);

        //绑定VAO绘制
        GLES30.glBindVertexArray(vaoId);
        //EBO索引绘制，6个索引
        GLES30.glDrawElements(GLES30.GL_TRIANGLES,6,GLES30.GL_UNSIGNED_INT,0);
        GLES30.glBindVertexArray(0);
    }

    //编译shader
    private int compileShader(int type,String src){
        int id = GLES30.glCreateShader(type);
        GLES30.glShaderSource(id,src);
        GLES30.glCompileShader(id);
        return id;
    }
    private int createProgram(String vert,String frag){
        int v = compileShader(GLES30.GL_VERTEX_SHADER,vert);
        int f = compileShader(GLES30.GL_FRAGMENT_SHADER,frag);
        int pro = GLES30.glCreateProgram();
        GLES30.glAttachShader(pro,v);
        GLES30.glAttachShader(pro,f);
        GLES30.glLinkProgram(pro);
        GLES30.glDeleteShader(v);
        GLES30.glDeleteShader(f);
        return pro;
    }
}
```

## 使用 FBO
默认渲染是画面直接画到手机屏幕（系统默认帧缓存），使用FBO (FrameBuffer Object)可以自定义一块离屏画布（GPU 上的纹理），先把画面渲染到它的纹理附件中（看不见），再拿这张纹理贴图绘制到屏幕。

```java
public class FBORenderer implements GLSurfaceView.Renderer {
    private static final int WIDTH = 1080;
    private static final int HEIGHT = 1920;

    // FBO 相关资源 ID
    private int fboId;
    private int textureId; // FBO 的纹理附件
    private int depthBufferId; // 深度缓冲（可选）

    // 屏幕渲染着色器程序（用于将 FBO 纹理绘制到屏幕）
    private int screenProgram;
    private int screenPositionHandle;
    private int screenTextureCoordHandle;
    private int screenTextureUniform;

    // 场景渲染着色器程序（渲染到 FBO）
    private int sceneProgram;
    private int scenePositionHandle;
    private int sceneColorHandle;

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        // 初始化 FBO 和纹理
        initFBO();
        
        // 编译着色器
        sceneProgram = buildSceneShaderProgram();
        screenProgram = buildScreenShaderProgram();
        
        // 设置视口
        GLES20.glViewport(0, 0, WIDTH, HEIGHT);
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GLES20.glViewport(0, 0, width, height);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        // === 步骤 1: 渲染到 FBO ===
        GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, fboId);
        GLES20.glViewport(0, 0, WIDTH, HEIGHT);
        
        // 清除 FBO 缓冲
        GLES20.glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT | GLES20.GL_DEPTH_BUFFER_BIT);
        
        // 使用场景着色器渲染 3D 物体（示例：绘制一个旋转的彩色三角形）
        GLES20.glUseProgram(sceneProgram);
        GLES20.glEnableVertexAttribArray(scenePositionHandle);
        GLES20.glEnableVertexAttribArray(sceneColorHandle);
        
        // 顶点数据（位置 + 颜色）
        float[] vertices = {
            -0.5f, -0.5f, 0.0f,  1.0f, 0.0f, 0.0f,  // 左下
             0.5f, -0.5f, 0.0f,  0.0f, 1.0f, 0.0f,  // 右下
             0.0f,  0.5f, 0.0f,  0.0f, 0.0f, 1.0f   // 顶部
        };
        ByteBuffer bb = ByteBuffer.allocateDirect(vertices.length * 4);
        bb.order(ByteOrder.nativeOrder());
        FloatBuffer vertexBuffer = bb.asFloatBuffer();
        vertexBuffer.put(vertices);
        vertexBuffer.position(0);
        
        // 传递顶点数据
        GLES20.glVertexAttribPointer(scenePositionHandle, 3, GLES20.GL_FLOAT, false, 24, vertexBuffer);
        GLES20.glVertexAttribPointer(sceneColorHandle, 3, GLES20.GL_FLOAT, false, 24, vertexBuffer.offset(12));
        GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, 3);
        
        // === 步骤 2: 将 FBO 纹理绘制到屏幕 ===
        GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, 0); // 解绑 FBO，切换回默认帧缓冲
        GLES20.glViewport(0, 0, WIDTH, HEIGHT);
        
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);
        GLES20.glUseProgram(screenProgram);
        
        // 传递全屏四边形顶点（覆盖整个屏幕）
        float[] screenQuad = {
            -1.0f, -1.0f, 0.0f, 0.0f,  // 左下
             1.0f, -1.0f, 1.0f, 0.0f,  // 右下
            -1.0f,  1.0f, 0.0f, 1.0f,  // 左上
             1.0f,  1.0f, 1.0f, 1.0f   // 右上
        };
        ByteBuffer bbScreen = ByteBuffer.allocateDirect(screenQuad.length * 4);
        bbScreen.order(ByteOrder.nativeOrder());
        FloatBuffer screenBuffer = bbScreen.asFloatBuffer();
        screenBuffer.put(screenQuad);
        screenBuffer.position(0);
        
        // 传递顶点数据
        GLES20.glVertexAttribPointer(screenPositionHandle, 2, GLES20.GL_FLOAT, false, 16, screenBuffer);
        GLES20.glVertexAttribPointer(screenTextureCoordHandle, 2, GLES20.GL_FLOAT, false, 16, screenBuffer.offset(8));
        GLES20.glEnableVertexAttribArray(screenPositionHandle);
        GLES20.glEnableVertexAttribArray(screenTextureCoordHandle);
        
        // 绑定 FBO 的纹理作为输入
        GLES20.glActiveTexture(GLES20.GL_TEXTURE0);
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureId);
        GLES20.glUniform1i(screenTextureUniform, 0);
        
        // 绘制全屏四边形
        GLES20.glDrawArrays(GLES20.GL_TRIANGLE_STRIP, 0, 4);
    }

    // 初始化 FBO 和纹理附件
    private void initFBO() {
        // 创建 FBO 对象
        int[] fbo = new int[1];
        GLES20.glGenFramebuffers(1, fbo, 0);
        fboId = fbo[0];
        
        // 创建纹理作为颜色附件
        int[] texture = new int[1];
        GLES20.glGenTextures(1, texture, 0);
        textureId = texture[0];
        
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureId);
        // 配置纹理参数
        GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MIN_FILTER, GLES20.GL_LINEAR);
        GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MAG_FILTER, GLES20.GL_LINEAR);
        GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_WRAP_S, GLES20.GL_CLAMP_TO_EDGE);
        GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_WRAP_T, GLES20.GL_CLAMP_TO_EDGE);
        // 分配空纹理内存（不传入像素数据）
        GLES20.glTexImage2D(GLES20.GL_TEXTURE_2D, 0, GLES20.GL_RGBA, WIDTH, HEIGHT, 0, GLES20.GL_RGBA, GLES20.GL_UNSIGNED_BYTE, null);
        
        // 创建深度缓冲（可选，但建议用于 3D 渲染）
        int[] depthBuffer = new int[1];
        GLES20.glGenRenderbuffers(1, depthBuffer, 0);
        depthBufferId = depthBuffer[0];
        
        GLES20.glBindRenderbuffer(GLES20.GL_RENDERBUFFER, depthBufferId);
        GLES20.glRenderbufferStorage(GLES20.GL_RENDERBUFFER, GLES20.GL_DEPTH_COMPONENT16, WIDTH, HEIGHT);
        
        // 将纹理和深度缓冲附加到 FBO
        GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, fboId);
        GLES20.glFramebufferTexture2D(GLES20.GL_FRAMEBUFFER, GLES20.GL_COLOR_ATTACHMENT0, GLES20.GL_TEXTURE_2D, textureId, 0);
        GLES20.glFramebufferRenderbuffer(GLES20.GL_FRAMEBUFFER, GLES20.GL_DEPTH_ATTACHMENT, GLES20.GL_RENDERBUFFER, depthBufferId);
        
        // 检查 FBO 状态
        int status = GLES20.glCheckFramebufferStatus(GLES20.GL_FRAMEBUFFER);
        if (status != GLES20.GL_FRAMEBUFFER_COMPLETE) {
            throw new RuntimeException("FBO not complete: " + status);
        }
        
        // 解绑 FBO
        GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, 0);
    }

    // 编译场景着色器（渲染到 FBO）
    private int buildSceneShaderProgram() {
        String vertexShaderCode =
            "attribute vec3 aPosition;" +
            "attribute vec3 aColor;" +
            "varying vec3 vColor;" +
            "void main() {" +
            "  gl_Position = vec4(aPosition, 1.0);" +
            "  vColor = aColor;" +
            "}";
        
        String fragmentShaderCode =
            "precision mediump float;" +
            "varying vec3 vColor;" +
            "void main() {" +
            "  gl_FragColor = vec4(vColor, 1.0);" +
            "}";
        
        int program = GLES20.glCreateProgram();
        int vs = GLES20.glCreateShader(GLES20.GL_VERTEX_SHADER);
        int fs = GLES20.glCreateShader(GLES20.GL_FRAGMENT_SHADER);
        
        GLES20.glShaderSource(vs, vertexShaderCode);
        GLES20.glShaderSource(fs, fragmentShaderCode);
        GLES20.glCompileShader(vs);
        GLES20.glCompileShader(fs);
        GLES20.glAttachShader(program, vs);
        GLES20.glAttachShader(program, fs);
        GLES20.glLinkProgram(program);
        
        // 获取属性位置
        scenePositionHandle = GLES20.glGetAttribLocation(program, "aPosition");
        sceneColorHandle = GLES20.glGetAttribLocation(program, "aColor");
        
        return program;
    }

    // 编译屏幕着色器（将 FBO 纹理绘制到屏幕）
    private int buildScreenShaderProgram() {
        String vertexShaderCode =
            "attribute vec2 aPosition;" +
            "attribute vec2 aTextureCoord;" +
            "varying vec2 vTextureCoord;" +
            "void main() {" +
            "  gl_Position = vec4(aPosition, 0.0, 1.0);" +
            "  vTextureCoord = aTextureCoord;" +
            "}";
        
        String fragmentShaderCode =
            "precision mediump float;" +
            "varying vec2 vTextureCoord;" +
            "uniform sampler2D uTexture;" +
            "void main() {" +
            "  gl_FragColor = texture2D(uTexture, vTextureCoord);" +
            "}";
        
        int program = GLES20.glCreateProgram();
        int vs = GLES20.glCreateShader(GLES20.GL_VERTEX_SHADER);
        int fs = GLES20.glCreateShader(GLES20.GL_FRAGMENT_SHADER);
        
        GLES20.glShaderSource(vs, vertexShaderCode);
        GLES20.glShaderSource(fs, fragmentShaderCode);
        GLES20.glCompileShader(vs);
        GLES20.glCompileShader(fs);
        GLES20.glAttachShader(program, vs);
        GLES20.glAttachShader(program, fs);
        GLES20.glLinkProgram(program);
        
        // 获取属性位置
        screenPositionHandle = GLES20.glGetAttribLocation(program, "aPosition");
        screenTextureCoordHandle = GLES20.glGetAttribLocation(program, "aTextureCoord");
        screenTextureUniform = GLES20.glGetUniformLocation(program, "uTexture");
        
        return program;
    }
}
```

# 为什么OpenGL可以在多个GLSurfaceView中正确绘制？
在EGL初始化以后，即渲染环境（EGLDisplay、EGLContext、GLSurface）准备就绪以后，需要在渲染线程（绘制图像的线程）中，明确的调用glMakeCurrent。这时，系统底层会将OpenGL渲染环境绑定到当前线程。
在这之后，只要你是在渲染线程中调用任何OpenGL ES的API（比如生产纹理ID的方法GLES20.glGenTextures），OpenGL会自动根据当前线程，切换上下文（也就是切换OpenGL的渲染信息和资源）。
换而言之，如果你在非调用glMakeCurrent的线程中去调用OpenGL的API，系统将找不到对应的OpenGL上下文，也就找不到对应的资源，可能会导致异常出错。
这也就是为什么有文章说，OpenGL渲染一定要在OpenGL线程中进行。

实际上，GLSurfaceView#Renderer的三个回调方法，都是在GLThread中进行调用的。