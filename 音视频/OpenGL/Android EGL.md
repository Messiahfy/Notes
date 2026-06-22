OpenGL ES 只是定义图形渲染API，还需要通过 EGL 来和设备实际的窗口系统交互。EGL（Embedded System Graphics Library）是嵌入式设备（比如Android）上 OpenGL ES 渲染 API 和本地窗口系统之间的接口层，由具体平台提供 EGL 的实现。Android 平台通过 EGL 给 OpenGL ES 搭建运行环境。

> 桌面平台（mac、win、linux）则有另外特定的窗口接口来提供 OpenGL 环境。

通俗地说，开发者使用OpenGL ES API调用EGL，在EGL的提供的包含Buffer的surface上绘制。EGL用在多个操作系统（如iOS、Android、Unix、Windows）和平台（包括X11、Windows这样的窗口系统，和GBM这样无显示器的渲染）。为了扩展性，EGL不依赖任何平台或渲染API的定义和概念，但有图形绘制需要的display等少数概念需要对应EGL规定的概念。


# 概念介绍

EGL作为OpenGL和平台窗口的中间层，包含`Display`、`Surface`、`context`：
* Display：代表实际显示设备
* Context：保存纹理 ID、Program、VAO、混合、深度测试等所有 GL 状态
* Surface：对用来存储图像的内存区域 FrameBuffer 的抽象，包括 Color Buffer、Stencil Buffer、Depth Buffer

## Surface
Surface 就是 EGL 层管理的帧缓存画布，分三大类：
1. WindowSurface：绑定系统窗口 → 输出屏幕
2. PbufferSurface：EGL 显存离屏画布 → 无窗口、不显示
3. PixmapSurface：系统内存位图画布 → 现代开发废弃

### WindowSurface
* 双缓冲设计（前缓冲 + 后缓冲），所有 OpenGL 绘制，默认画在后台缓冲 (BackBuffer)
* 调用 eglSwapBuffers(Display,Surface) 交换前后缓冲区，后台画面送到前台屏幕显示，如果不调用则屏幕永远不变，绘制内容藏在后台出不来
* 不能直接拿缓冲区像素；要取图就需要使用 glReadPixels

> 适用于 APP 画面渲染、相机预览、播放器、3D 游戏等场景

### PbufferSurface
PBuffer 就是 Pixel Buffer（显存像素缓冲区），也就是无系统窗口，完全在显存开辟一块自定义宽高的画布
* 没有屏幕关联，画面永远不可见
* 禁止调用 eglSwapBuffers，调用直接报错 EGL_BAD_SURFACE，没有双缓冲交换逻辑
* 绘制完使用 glReadPixels 读取 RGBA 像素到内存

> 适用于后台生成图片、图像处理、滤镜合成、OpenGL 出裸数据流送给硬编码器。

> ⚠️ 缺点：单独创建 PBuffer 会额外占用一块显存，现代优化：只用 WindowSurface + FBO 做离屏，减少 Surface 创建

> FBO 等概念在 OpenGL 中介绍

# API 使用
官方文档：https://registry.khronos.org/EGL/sdk/docs/man/

## 一般使用流程

1. 获取 EGL Display 对象作为渲染目标：eglGetDisplay()
2. 初始化与 EGLDisplay 之间的连接：eglInitialize()
3. 获取 EGLConfig，确定配置信息：eglChooseConfig()
4. 创建 EGLContext ：eglCreateContext()
5. 创建 EGLSurface：eglCreateWindowSurface()/eglCreatePbufferSurface()
6. 关联 EGLContext 和 EGLSurface：eglMakeCurrent() **到这一步就完成了环境创建，下面可以开始渲染**
7. 使用 `OpenGL ES API` 配合 shader 绘制图形：gl_*()
8. 使用 WindowSurface 的情况切换 front buffer 和 back buffer 送显：eglSwapBuffer()
9. 释放资源：eglDestroySurface、eglDestroyContext、eglTerminate（对应eglInitialize）

## 核心流程函数

### eglGetDisplay
// C 代码为： EGLDisplay eglGetDisplay (EGLNativeDisplayType display_id)
public static native EGLDisplay eglGetDisplay(int display_id);

* 获取和display_id关联的EGL Display
* EGL_DEFAULT_DISPLAY对应默认的显示设备
* 使用相同的display_id 多次调用eglGetDisplay 将返回相同的 EGLDisplay 句柄
* 如果没有和display_id匹配的Display，将返回EGL_NO_DISPLAY，可以用EGL_NO_DISPLAY判断失败
* 需要调用`eglInitialize`函数初始化display连接。

### eglInitialize
```c
// C 代码为： EGLBoolean eglInitialize (EGLDisplay dpy, EGLint *major, EGLint *minor)

public static native boolean eglInitialize(
    EGLDisplay dpy, // 传入eglGetDisplay获取的EGLDisplay
    int[] major, // 这是一个输出参数，传入数组，调用函数后数组内将存入EGL的主版本号
    int majorOffset, // 存到数组的哪个位置
    int[] minor, // 和major一样是输出参数，存入EGL的副版本号
    int minorOffset // 存到数组的哪个位置
);
```

```c
// 调用示例
int[] version = new int[2];
if(!mEgl.eglInitialize(mEglDisplay, version, 0, version, 1)) {
    throw new RuntimeException("eglInitialize failed");
}
```

* 初始化 EGL display 连接
* 除了返回版本号之外，初始化已经初始化的 EGL Display连接没有任何作用
* EGL_FALSE、EGL_BAD_DISPLAY、EGL_NOT_INITIALIZED表示初始化失败的几种情况，但Android返回boolean，false应该就对应各种失败了，要具体检测错误类型的话，可以用 `eglGetError`
* 调用`eglTerminate`可以释放和EGL display 连接 关联的资源

### eglChooseConfig
```c
// C 代码函数： EGLBoolean eglChooseConfig (EGLDisplay dpy, const EGLint *attrib_list, EGLConfig *configs, EGLint config_size, EGLint *num_config)

public static native boolean eglChooseConfig(
    EGLDisplay dpy,
    int[] attrib_list, // 指定配置所需匹配的属性，也就是需要的配置
    int attrib_listOffset, // 存到数组的哪个位置，偏移参数
    EGLConfig[] configs, // 输出参数
    int configsOffset,
    int config_size, // 最多只有 config_size 这么多的EGLConfig存到configs中返回给我们
    int[] num_config, // 输出参数，表示匹配的全部EGLConfig数量
    int num_configOffset // 数据存到 num_config 的哪个位置，一般num_config使用长度为1的数组，num_configOffset传0即可
);
```

* 返回Display与指定属性（放在attrib_list中）匹配的 EGL 帧缓冲区（EGLSurface）配置列表，放在configs中输出
* 可以看GLSurfaceView中的调用方式了解各参数用法：先configs传null，config_size传0，通过num_config得到匹配配置的数量，再根据全部数量，传入了configs和config_size，从而用刚好是全部数量的configs数组大小，去得到全部EGLConfig，再遍历全部EGLConfig调用`eglGetConfigAttrib`找到符合全部配置的第一个EGLConfig
* 如果attrib_list中没有定义某个属性，该属性将使用默认值。例如 EGL_DEPTH_SIZE 没有设置，将会是0
* 某些属性的默认值为EGL_DONT_CARE，表示任何值都可以，所以该属性不会被检测
* 每个属性的匹配方式可能不一样，比如EGL_LEVEL必须刚好相等，而EGL_RED_SIZE必须大于等于指定的值才能匹配
* 如果有多个匹配全部属性的EGLConfig，则EGLConfig会根据最佳匹配标准排序，所以很多用法是


attrib_list参数的使用方式为key-value顺序排列的数组，并以 EGL_NONE 结尾：
```java
new int[] {
    EGL10.EGL_RED_SIZE, redSize,
    EGL10.EGL_GREEN_SIZE, greenSize,
    EGL10.EGL_BLUE_SIZE, blueSize,
    EGL10.EGL_ALPHA_SIZE, alphaSize,
    EGL10.EGL_DEPTH_SIZE, depthSize,
    EGL10.EGL_STENCIL_SIZE, stencilSize,
    EGL10.EGL_NONE}
```

介绍一些常用的配置，更完整的可以查看EGL官方文档
* EGL_ALPHA_SIZE：设置颜色buffer的alpha占的bit位大小，默认为0，一般传8或0，其他值一般找不到匹配的配置
* EGL_RED_SIZE：设置颜色buffer的red占的bit为大小，一般传8，其他值一般找不到匹配的配置
* EGL_GREEN_SIZE：设置颜色buffer的green占的bit为大小，一般传8，其他值一般找不到匹配的配置
* EGL_BLUE_SIZE：设置颜色buffer的blue占的bit为大小，一般传8，其他值一般找不到匹配的配置
* EGL_DEPTH_SIZE：设置深度buffer大小，默认为0，必须为非负数。
* EGL_STENCIL_SIZE：设置stencil 目标 buffer大小，默认为0

* EGL_BIND_TO_TEXTURE_RGB：设置EGL_DONT_CARE、EGL_TRUE、EGL_FALSE其中一个，默认为EGL_DONT_CARE。如果设置为EGL_TRUE，则仅考虑支持将颜色缓冲区绑定到 OpenGL ES RGB 纹理的帧缓冲区配置。 目前只有支持 pbuffer 的帧缓冲区配置允许。
* EGL_BIND_TO_TEXTURE_RGBA：设置EGL_DONT_CARE、EGL_TRUE、EGL_FALSE其中一个，默认为EGL_DONT_CARE。如果设置为EGL_TRUE，则仅考虑支持将颜色缓冲区绑定到 OpenGL ES RGBA 纹理的帧缓冲区配置。 目前只有支持 pbuffer 的帧缓冲区配置允许。
* EGL_BUFFER_SIZE：EGL_RED_SIZE、EGL_GREEN_SIZE、EGL_BLUE_SIZE、EGL_ALPHA_SIZE的值总和，一般不怎么用这个，基本是单独设置各个颜色的bit占用大小
* EGL_COLOR_BUFFER_TYPE：可选EGL_RGB_BUFFER 或 EGL_LUMINANCE_BUFFER。EGL_RGB_BUFFER表示RGB颜色缓冲区，这种情况下，属性 EGL_RED_SIZE、EGL_GREEN_SIZE 和 EGL_BLUE_SIZE 必须为非零，并且 EGL_LUMINANCE_SIZE 必须为零。EGL_LUMINANCE_BUFFER 表示亮度颜色缓冲区。这种情况下，EGL_RED_SIZE、EGL_GREEN_SIZE、EGL_BLUE_SIZE 必须为零，并且 EGL_LUMINANCE_SIZE 必须为非零。两种情况对 EGL_ALPHA_SIZE都不做要求
* EGL_CONFORMANT：用于指定所需的 OpenGL ES 版本和功能级别，以确保应用程序在特定的设备上能够正确运行
* EGL_LUMINANCE_SIZE：指示颜色缓冲区的亮度分量所需的大小，以位为单位
* EGL_TRANSPARENT_TYPE：透明帧缓冲区配置

* EGL_RENDERABLE_TYPE：用于指定可渲染的图形 API 类型。它描述了eglCreateContext创建的context必须支持的图形 API 类型。
* EGL_SURFACE_TYPE：设置frame buffer配置必须支持的EGL surface类型。EGL_WINDOW_BIT对应window suface、EGL_PBUFFER_BIT对应pixel buffer surface，EGL_PIXMAP_BIT对应 pixmap surface，未列举全部，其他类型可看官方文档。默认为EGL_WINDOW_BIT。


错误返回值：
* EGL_FALSE：表示失败，EGL_TRUE为成功。如果返回EGL_FALSE，则configs和num_config没有被修改
* EGL_BAD_DISPLAY：表示display并不是一个EGL display connection
* EGL_BAD_ATTRIBUTE：如果attribute_list包含无效、错误配置。
* EGL_NOT_INITIALIZED：如果display没有被初始化
* EGL_BAD_PARAMETER：如果num_config为null

### eglCreateContext
```c
// C代码：
EGLContext eglCreateContext(EGLDisplay display,
 	EGLConfig config,
 	EGLContext share_context,
 	EGLint const * attrib_list);
```

```java
// android 方法
public static native EGLContext eglCreateContext(
    EGLDisplay dpy,
    EGLConfig config, // 通过eglChooseConfig得到的配置
    EGLContext share_context,
    int[] attrib_list, // 设置用于EGLContext创建的属性列表
    int offset
);
```
* eglCreateContext函数会创建一个EGL渲染上下文用于当前渲染API，上下文可用于渲染到EGL的绘制surface。如果创建失败将返回EGL_NO_CONTEXT
* share_context：如果不是EGL_NO_CONTEXT，则context中可以共享的数据将被共享。如果该 context 之前已经与其他 context 共享，那么它们多个之间，都可以共享。比如共享纹理、program等。

attrib_list可以使用的配置：
* EGL_CONTEXT_MAJOR_VERSION：OpenGL ES的context的主版本，默认为1。也可以使用EGL_CONTEXT_CLIENT_VERSION。
* EGL_CONTEXT_MINOR_VERSION：OpenGL ES的context的副版本，默认为0.
* EGL_CONTEXT_OPENGL_PROFILE_MASK：设置OpenGL context的profile，默认为EGL_CONTEXT_OPENGL_CORE_PROFILE_BIT，表示EGL context使用默认配置文件；EGL_CONTEXT_OPENGL_COMPATIBILITY_PROFILE_BIT表示使用Compatibility 配置文件。
* EGL_CONTEXT_OPENGL_DEBUG：设置是否创建debug context，默认为EGL_FALSE。debug context情况下，OpenGL 的错误检查和调试功能将打开，可以帮助开发者在运行时检测和调试应用程序中的 OpenGL 错误。打开 OpenGL 调试可能会对应用程序的性能产生一定的影响。
* EGL_CONTEXT_OPENGL_FORWARD_COMPATIBLE：指定是否创建向前兼容的context，默认为EGL_FALSE。在向前兼容的上下文中，将忽略一些过时的 OpenGL 功能和特性，以便使用更新的 OpenGL 版本或扩展。但可能会限制使用某些过时的 OpenGL 功能和扩展。在向前兼容的上下文中，某些功能可能被强制启用或禁用，具体取决于 OpenGL 实现的支持情况。可能会导致一些不兼容的行为，特别是当应用程序依赖于过时功能或特定的 OpenGL 版本特性时。
* EGL_CONTEXT_OPENGL_ROBUST_ACCESS：设置context是否支持robust buffer access。robust buffer access是指 OpenGL 实现在面对一些异常情况时能够保证正确而可控的行为，以保护应用程序免受不正确的操作或错误状态的影响。
* EGL_CONTEXT_OPENGL_RESET_NOTIFICATION_STRATEGY：设置openGL发生重置时的通知策略。EGL_NO_RESET_NOTIFICATION表示不接收重置通知；EGL_LOSE_CONTEXT_ON_RESET表示希望在 OpenGL 重置时丢失上下文，应用程序需要重新创建上下文，并重新初始化 OpenGL 对象和状态。


> 创建好Context后，环境也就搭建完毕，下面就需要将EGL和设备的屏幕连接起来。

### eglCreateWindowSurface / eglCreatePbufferSurface / eglCreatePlatformPixmapSurface
```c
// C函数代码
EGLSurface eglCreateWindowSurface(EGLDisplay display,
 	EGLConfig config,
 	NativeWindowType native_window,
 	EGLint const * attrib_list);
```

```java
// android 方法
public static EGLSurface eglCreateWindowSurface(EGLDisplay dpy,
    EGLConfig config, // 仍然传入eglChooseConfig中得到的配置即可
    Object win, // Android中可以传入Surface
    int[] attrib_list,
    int offset
)
```
* 创建 on-screen EGL window surface
* attrib_list：可以是NULL或者第一个属性为EGL_NONE的数组，这种情况就会使用默认值

```c
// C函数代码
EGLSurface eglCreatePbufferSurface(	EGLDisplay display,
 	EGLConfig config,
 	EGLint const * attrib_list);
```
```java
public static native EGLSurface eglCreatePbufferSurface(
    EGLDisplay dpy,
    EGLConfig config,
    int[] attrib_list,
    int offset
);
```
* 创建 off-screen pixel buffer surface，如果失败将返回EGL_NO_SURFACE


attrib_list可以设置如下值：
* EGL_WIDTH：设置pixel buffer surface的宽度，默认为0
* EGL_HEIGHT：设置pixel buffer surface的高度，默认为0
* EGL_GL_COLORSPACE：指定渲染到surface时 OpenGL 和 OpenGL ES 使用的颜色空间，默认为EGL_GL_COLORSPACE_LINEAR
* EGL_LARGEST_PBUFFER：尝试创建最大尺寸的 Pbuffer Surface，以适应当前设备的可用屏幕空间。 使用eglQuerySurface 检索已分配的Pbuffer Surface的尺寸。 默认值为 EGL_FALSE。
* EGL_MIPMAP_TEXTURE：是否为 mipmap 分配存储空间
* EGL_TEXTURE_FORMAT：设置当 pbuffer 绑定到texture map时将创建的texture格式，可以使用 EGL_NO_TEXTURE、EGL_TEXTURE_RGB 和 EGL_TEXTURE_RGBA。 默认值为 EGL_NO_TEXTURE。通过指定 EGL_TEXTURE_FORMAT 属性，可以告诉 EGL 在创建图像纹理时使用的颜色格式。这使得 EGL 和 OpenGL ES 可以正确地解释和处理纹理数据，并在渲染过程中进行正确的颜色显示。
* EGL_TEXTURE_TARGET：设置当使用 EGL_TEXTURE_RGB 或 EGL_TEXTURE_RGBA 纹理格式创建 pbuffer 时将创建的纹理的目标。 可能的值为 EGL_NO_TEXTURE 或 EGL_TEXTURE_2D。 默认值为 EGL_NO_TEXTURE。通过指定 EGL_TEXTURE_TARGET 属性，可以告诉 EGL 纹理将用于哪个图形 API 或扩展
* EGL_VG_ALPHA_FORMAT：设置OpenVG如何使用alpha值
* EGL_VG_COLORSPACE：OpenVG用于渲染的颜色空间

----------------------------

**与绘图表面相关的概念如下**：
1. Window Surface：
Window Surface绑定操作系统原生窗口，用于画面输出到屏幕。WindowSurface**硬件规范固定双缓冲**，自带Front、Back缓冲区，必须调用eglSwapBuffers完成前后缓冲交换，画面才能刷新到屏幕。

2. Pbuffer Surface：
Pbuffer Surface是GPU上创建的离屏绘图表面，不关联屏幕。Pbuffer**固定单缓冲架构，不存在Front/Back双缓冲，不支持eglSwapBuffers**，渲染结果保存在唯一缓冲中，使用glReadPixels读取像素数据，用于离线渲染、图片导出、编码推流。

**关系**：
Back Buffer / Single Buffer 是缓冲区存储规格；Window/Pbuffer是EGL画布载体。
- WindowSurface：固定双缓冲，绘图到Back，Swap交换至Front上屏；
- PbufferSurface：固定单缓冲，无交换逻辑，绘图后直接读像素；
二者不能自由切换单/双缓冲模式。

#### eglCreatePlatformPixmapSurface（使用较少）
```c
EGLSurface eglCreatePlatformPixmapSurface(EGLDisplay display,
 	EGLConfig config,
 	void * native_pixmap,
 	EGLint const * attrib_list);
```
创建off-screen pixmap surface

> Pixmap是操作系统原生内存位图（X11/Windows GDI 位图），用于在一段内存位图上渲染，不在显存上，性能差。Android不支持使用。


### eglMakeCurrent
```c
// C函数代码
EGLBoolean eglMakeCurrent(EGLDisplay display,
 	EGLSurface draw,
 	EGLSurface read,
 	EGLContext context);
```

```java
// android 方法
public static native boolean eglMakeCurrent(
    EGLDisplay dpy,
    EGLSurface draw,
    EGLSurface read,
    EGLContext ctx
);
```
* 绑定context到当前的渲染线程和draw、read surface。调用eglMakeCurrent之后，才可以调用 OpenGL ES 的API修改context和操作surface。
* draw surface可以用于除`glReadPixels`、`glCopyTexImage2D` 和 `glCopyTexSubImage2D` 外的所有操作，因为这几个操作是从read surface的frame buffer读取数据。也就是draw surface用于除读取和复制外的操作，read surface用于读取和复制。
* draw和read可以是同一个EGLSurface，一般也会设置为同一个。
* 如果调用线程已经有和相同绘制API的context，那么已有的context将被flushed并且标记为当前未使用。新的context将成为调用线程的当前context。出于 eglMakeCurrent 的目的，所有OpenGL ES 和OpenGL 上下文的客户端API 类型被视为相同，换句话说，只要当前有context，就会替换为新的context。
* 如果调用eglMakeCurrent后draw surface被销毁，那么后续的渲染命令将被处理并且context状态将被更新，但surface内容未定义。如果在eglMakeCurrent之后read被销毁，则从帧缓冲区读取的像素值（例如调用glReadPixels的结果）是未定义的。
* 要释放当前context并且不设置新的context，则设置context为EGL_NO_CONTEXT，并设置draw和read为EGL_NO_SURFACE，当前context将flushed并标记为未使用。
* 首次设置context，viewport和scissor将设置为draw surface的大小，但更换context后，需要自行重新设置viewport和scissor的大小


> 调用eglMakeCurrent后，为 OpenGL ES 渲染提供了目标 EGLDisplay 及上下文环境 EGLContext，也连接了EGLSurface。之后的 OpenGL ES 渲染需要和eglMakeCurrent的调用在同一个线程

### eglSwapBuffer
```c
// C函数代码
EGLBoolean eglSwapBuffers(EGLDisplay display,
 	EGLSurface surface);
```

```java
public static native boolean eglSwapBuffers(
    EGLDisplay dpy,
    EGLSurface surface
);
```
* 如果surface是back-buffered window surface，则color buffer见被复制到关联此surface的屏幕上；如果surface是single-buffered window或者pixel buffer surface，eglSwapBuffers调用没有效果。
* eglSwapBuffers 在交换之前对绑定到surface的上下文执行隐式flush操作（对于 OpenGL ES 或 OpenGL 上下文为 glFlush，对于 OpenVG 上下文为 vgFlush）。调用eglSwapBuffers 后，可以立即在该上下文上发出后续的客户端 API 命令，但直到缓冲区交换完成后才会执行。

完成渲染后，可以调用eglSwapBuffers，以将当前绑定到back-buffered window surface的渲染结果交换到屏幕上。如果是single-buffered
window, pixmap, 或者 pbuffer surface，此方法无效。


## 其他函数

### eglDestroySurface
```java
public static native boolean eglDestroySurface(
    EGLDisplay dpy,
    EGLSurface surface
);
```
* 如果surface没有绑定到任何线程，eglDestroySurface立即完成；否则在和所有线程解绑后销毁（所以要先调eglMakeCurrent传入EGL_NO_SURFACE）。并且pbuffer surface关联的资源会在绑定到texture释放后才能释放（所以需要释放纹理）


### eglDestroyContext
```java
public static native boolean eglDestroyContext(
    EGLDisplay dpy,
    EGLContext ctx
);
```
* 销毁context。如果context没有绑定任何线程，eglDestroyContext立即完成。否则，context将在和所有绑定线程解绑后销毁（也就是要先调用eglMakeCurrent传入EGL_NO_CONTEXT解绑）。

### eglTerminate
// C语言：EGLBoolean eglTerminate(EGLDisplay display);

标记与指定Display相关联的所有 EGL 特定资源（例如context和surface）以供删除。 eglTerminate 返回后，所有此类资源的句柄都将无效，但 display 句柄本身仍然有效。如果 display 关联的 context 或者 surfaces 被其他线程调用了 eglMakeCurrent 绑定，则会知道其他线程解绑后才会销毁。

可以再次调用 eglInitialize 重新初始化 display。


### eglGetCurrentContext、eglGetCurrentDisplay、eglGetCurrentSurface
* eglGetCurrentContext 函数用于获取当前线程的当前上下文。它返回当前线程正在使用的OpenGL ES上下文的句柄。如果该线程没有绑定上下文，则返回 EGL_NO_CONTEXT。
* eglGetCurrentDisplay 返回当前线程的context关联的EGL display，和eglMakeCurrent相关。如果当前线程没有设置context，则返回EGL_NO_DISPLAY。
* eglGetCurrentSurface 返回当前线程的context关联的read或draw surface，如果当前线程没有设置context，则返回EGL_NO_SURFACE


### eglGetConfigAttrib eglGetConfigs
* eglGetConfigAttrib返回属性值
* eglGetConfigs返回EGL display可用的全部属性配置

### eglQueryContext eglQueryString eglQuerySurface
* eglQueryContext：查询context中的一些属性值，比如EGL_CONTEXT_CLIENT_VERSION、EGL_RENDER_BUFFER
* eglQueryString：查询与 EGL 实现相关的信息。它可以用来获取 EGL 版本、扩展支持、客户端 API 等信息。
* eglQuerySurface：查询和EGL Surface相关的属性，比如尺寸、像素格式等信息

### eglBindAPI 和 eglQueryAPI
eglBindAPI 设置当前线程的绘制 API，例如EGL_OPENGL_ES_API、EGL_OPENGL_API、EGL_OPENVG_API。会影响一些api的行为。

eglQueryAPI 查询当前线程的绘制API

### eglBindTexImage 和 eglReleaseTexImage
```java
public static native boolean eglBindTexImage(
    EGLDisplay dpy,
    EGLSurface surface, // 纹理将和这个EGLSurface绑定
    int buffer // 一般填 EGL_BACK_BUFFER
);
```

* eglBindTexImage：把某个 EGLSurface 的颜色缓冲区，直接 “借给” OpenGL ES 纹理用，零拷贝、直接共享显存。
* eglReleaseTexImage：用完了，把缓冲区还给 Surface，让 Surface 恢复正常读写。

> 核心场景：把 Surface 渲染结果直接当纹理用，性能更好。但 Surface 暂时变成一块只读纹理，不能再绘制。



### eglCreateSync  eglDestroySync  eglClientWaitSync  eglWaitSync eglGetSyncAttrib
```java
public static native EGLSync eglCreateSync(
    EGLDisplay dpy,
    int type, // 要创建的sync的类型
    long[] attrib_list, // 设置sync的属性
    int offset
);
```
* sync对象用于让 CPU / 其他上下文 / 其他进程知道：GPU 前面这堆命令什么时候跑完了
* sync对象有两种可能的状态：signaled和unsignaled。同步对象初始是unsignaled。EGL 可能会被要求等待sync对象状态变为signaled，或者可能会查询sync对象的状态。
* 一旦满足sync对象的条件，将变为signaled，从而使阻塞的eglClientWaitSync 或 eglWaitSync 解除阻塞。
* 如果type是EGL_SYNC_FENCE，将创建 fence sync 对象。这种情况，attrib_list必须为NULL或者仅包含EGL_NONE的数组。创建 fence sync 对象时，eglCreateSync还会插入一个fence命令到绑定的上下文的命令流中。
* 如果type是EGL_SYNC_CL_EVENT，将创建 OpenCL event sync 对象。

> 作用：GPU 是异步的，CPU 提交 gl 命令 → 放到 GPU 队列 → CPU 立刻返回继续跑，不知道 GPU 什么时候真的画完。eglCreateSync 就是插一个 “标记”，CPU / 其他线程 / 其他进程可以等待这个信号。

```java
public static native int eglClientWaitSync(
    EGLDisplay dpy,
    EGLSync sync,
    int flags,
    long timeout
);
```
* eglClientWaitSync将阻塞调用线程直到sync对象变为signaled，或者超时
* 当多个线程在同一个sync对象上阻塞，并且sync对象变为signaled，这些线程都会被解除阻塞，但顺序未定义。
* timeout如果为0，eglClientWaitSync仅测试sync的当前状态；如果为EGL_FOREVER，则eglClientWaitSync不会超时；其他值则为作为正常超时来使用。
* 如果返回EGL_TIMEOUT_EXPIRED，表示已超时，或者timeout为0表示sync对象还没有变为signaled
* 如果返回EGL_CONDITION_SATISFIED，表示在超时到期之前发出了sync对象状态已变为signaled，包括调用eglClientWaitSync 时已经发出同步信号的情况。
* 如果返回EGL_FALSE，表示发生错误
* flags如果设置EGL_SYNC_FLUSH_COMMANDS_BIT，等待同步对象的同时会flush渲染管道中的命令。这可以确保在等待期间，之前提交的渲染命令已经执行完毕。


eglClientWaitSync 和 eglWaitSync 的区别：
* `eglClientWaitSync`：CPU 线程阻塞等待，直到 sync 被触发或超时；能拿到返回状态（成功 / 超时 / 失败）。
* `eglWaitSync`：GPU 端等待，CPU 立刻返回；把 “等 sync” 这个动作交给 GPU 命令流，CPU 不阻塞、不卡线程。

### eglCopyBuffers
```java
public static native boolean eglCopyBuffers(
    EGLDisplay dpy,
    EGLSurface surface, 从这个surface复制
    int target // 复制数据放到这里
);
```
* 复制surface中的color buffer到target（pixmap指针）中
* eglCopyBuffers会在返回前隐式执行glFlush，返回的后续GL命令会直到复制完成后才执行。


### eglCreateImage eglDestroyImage
```java
public static native EGLImage eglCreateImage(
    EGLDisplay dpy,
    EGLContext context,
    int target,
    long buffer,
    long[] attrib_list,
    int offset
);
```
* 用于创建 EGLImage 对象
* context参数指定用于此操作的EGL client API context，如果不需要则使用EGL_NO_CONTEXT
* target指定作为EGLImage输入源的资源类型，比如EGL_GL_TEXTURE_2D、EGL_GL_RENDERBUFFER
* buffer是作为EGLImage输入源的名称或句柄

> EGLImage：绑定系统 NativeBuffer/AHardwareBuffer 硬件内存做纹理，直接把硬件缓冲区映射成 GL 纹理，全程无拷贝。

### eglGetError
在每个egl函数调用后，可以调用eglGetError获取是否发生错误

### eglReleaseThread
相当于eglMakeCurrent传入EGL_NO_CONTEXT和EGL_NO_SURFACE，并且重置eglBindAPI，其他基于线程维护的状态都被删除，比如eglGetError相关的错误状态。恢复到线程的初始EGL状态

### eglSurfaceAttrib
设置 EGL surface 的属性，比如EGL_MIPMAP_LEVEL、EGL_MULTISAMPLE_RESOLVE、EGL_SWAP_BEHAVIOR

### eglSwapInterval
决定了在双缓冲渲染中，绘制到前缓冲的图像在切换到屏幕上之前需要等待的帧数。将影响eglSwapBuffers。默认为1，1相当于垂直同步，也就是在下一次屏幕刷新之前等待一个帧；如果是0就相当于禁用垂直同步，swap会在渲染完成后尽快执行。

传入的interval，会限制在 EGLConfig 属性 EGL_MIN_SWAP_INTERVAL 和 EGL_MAX_SWAP_INTERVAL 之间。

### eglWaitClient
阻塞当前线程，直到之前调用的客户端 API （包含OpenGL ES 和其他与 EGL 相关的客户端 API）执行完成为止。`glFinish`也可以达到一样的效果。

### eglWaitGL
阻塞当前线程，直到之前调用的OpenGL ES API 的完成

### eglWaitNative
等待与 EGL 显示连接关联的本地平台事件。当 EGL 显示连接与本地平台的窗口系统进行交互时，例如窗口重绘、输入事件等，可以使用 eglWaitNative 函数来等待这些本地平台事件的发生。


# 2. EGL 操作

## 2.1 本地平台和渲染API

### EGL 类型
* `EGLBoolean`：只有`EGL_TRUE`和`EGL_FALSE`两个值。如果其他的值作为boolean参数传给EGL，行为未定义，即使通常情况非零值会被当做`EGL_TRUE`。
* `EGLint`：整型
* `EGLAttrib`：和 intptr_t（指针）类型一样。用于eglCreateImage、eglGetPlatformDisplay等命令，并且会被用于未来所有需要attribute参数和返回attribute列表的相似命令。
* `EGLContext`：表示客户端API的context。context的定义取决于客户端 API，但通常表示客户端 API 的状态机，可以执行与该状态机相关的客户端 API 命令。
* `EGLImage`：是一个用于在不同的图形 API 之间共享图像数据。它提供了一种机制，使得图像数据可以在 OpenGL ES、OpenVG、OpenCL 和其他图形 API 之间高效地共享，而无需进行昂贵的数据拷贝操作。
* `EGLSurface`：渲染的目标缓冲区
* `EGLSync`：用于同步 GPU 和 CPU 的操作
* `EGLTime`：用于指定同步操作的超时时间

### Displays
大多数EGL调用包含 `EGLDisplay` 参数，它表示用于图形绘制的抽象显示器，大多数情况，display对应一个物理显示器。所有EGL对象都会关联 `EGLDisplay`，并存在于该display定义的命名空间。所有EGL对象被`EGLDisplay`参数和表示对象句柄参数的组合来指定。

## 2.2 Rendering Contexts 和 Drawing Surfaces
client API（如OpenGL ES）规范故意对如何创建渲染context（client API 的状态机）没有具体定义。EGL的目的之一就是提供一种创建客户端 API 渲染上下文（简称为上下文）并将其与绘图surface关联的方法。

EGL 定义了几种类型的绘图surface，统称为 `EGLSurface`。包括用于屏幕渲染的`window`，用于离屏渲染的`pbuffer`，和用于离屏渲染到可以通过native API 访问的缓冲区中的`pixmap`。EGL `window`和`pixmap`与平台的`window`和`pixmap`相关联。

> native 渲染API是特定于操作系统或图形库的底层接口，用于直接与底层图形硬件和渲染管线交互。而Client API（比如 OpenGL ES）提供了提供了独立于硬件和操作系统的标准化接口，是更高级的抽象和封装，使开发人员更方便地进行图形渲染。EGL就是它们的中间层。

`EGLSurfaces`会根据 `EGLConfig` 配置创建，`EGLConfig`描述了color buffer（颜色缓冲区）的位深，和ancillary缓冲区（比如 位深、multisample、stencil buffers）的类型、数量和大小。ancillary缓冲区和 `EGLSurface` 而不是context关联，如果几个context写入同一个surface，它们会共享缓冲区。

不同client APIs的context共享surface的颜色缓冲区，但ancillary缓冲区对于每个cient API没有必要的意义。context可以用于任何兼容的`EGLSurfaces`，满足以下条件即兼容：
* 支持同样的颜色缓冲区类型（RGB或luminance）
* 颜色缓冲区和ancillary buffers有同样的位深
* surface对应的EGLConfig支持与上下文的 API 类型相同类型的客户端 API 渲染（在支持多个客户端 API 的环境中）。
* 它们是针对相同的 EGLDisplay 创建的（在支持多个display的环境中）。

只要满足兼容性约束和地址空间要求，客户端就可以使用不同的上下文渲染到相同的 EGLSurface 中。 还可以使用单个上下文渲染到多个 EGLSurface 中。

### 使用渲染上下文
OpenGL 和 OpenGL ES context 由两个部分组成：一个持有client状态，一个持有server状态。OpenGL和OpenGL ES的API隐式依赖context（即 current context）。

每个线程可以有最多一个 current context，一个context在同一时间只能作为一个线程的current contex。

###  和 Native Rendering 交互
Native Render：原生 API 渲染：指不用 OpenGL ES，用平台原生绘图 API（Windows GDI、Linux X11/Xlib）直接在同一块缓冲区画画

pixmap surface 支持native rendering，pixmap surface通常用于希望混合native和client API渲染时，因为这样不用在back buffer 和 client API之间、 native pixmap 和 native rendering API 之间移动数据。但也因为这样，相对于window和pbuffer surface，pixmap surface 的能力和性能受到限制。

pbuffer surface不支持 native rendering，因为pbuffer的颜色缓冲区被EGL内部分配，不能通过其他方式访问。

window surface可能支持 native rendering，但仅在平台有允许共享 back color buffer这样的兼容性渲染模式，或者正进行对window surface的single buffered 渲染。

## 2.3 直接渲染和地址空间
EGL仅支持直接渲染。EGL对象和context相关的状态不能被用于创建它们的地址空间之外。在单线程环境，每个进程拥有自己的地址空间；在多线程环境，所有线程共享虚拟地址空间；但这种能力并不是必需的，即使在支持多个线程的环境中，实现也可以选择将其地址空间限制为单个线程。

上下文状态包括OpenGL和OpenGL ES的client和server状态，存在于客户端的地址空间中；此状态不能由另一个进程中的客户端共享。

## 2.4 共享状态
大多数context状态都很小，但某些类型的状态可能很大和复制消耗大，在这种情况下，可能需要多个context共享这些状态而不是复制到每个context中。这种状态只能在相同API类型的上下文中共享（比如都是OpenGL ES），EGL 提供在单个地址空间中存在的上下文之间共享某些类型的上下文状态。可共享的client API 对象的类型由相应的客户端 API 规范定义。

### 2.4.1 OpenGL 和 OpenGL ES 纹理对象
纹理状态封装在纹理对象中。OpenGL 和 OpenGL ES 不会尝试同步对纹理对象的访问。如果纹理对象绑定到多个上下文，则程序员需要确保在一个上下文使用该纹理对象进行渲染时，该对象的内容不会通过另一个上下文进行更改。 当另一个上下文正在使用纹理对象时更改纹理对象的结果是未定义的。

执行 glBindTexture 对共享上下文状态的所有修改都是原子的。此外，当纹理对象仍绑定到任何上下文时，它不会被删除。

### 2.4.2 OpenGL 和 OpenGL ES Buffer对象
Buffer对象和纹理对象的并发行为类似

## 2.5 EGLImage
在某些情况下，可能需要在客户端 API 之间共享状态。一个示例是使用先前渲染的 OpenVG 图像作为 OpenGL ES 纹理对象。EGLImage可以实现这样的情况，

client api提供了从 EGLImage 创建需要的资源类型（比如纹理数组或OpenVG的VGImage）的机制。使用客户端 API 通过从 EGLImage 创建的资源称为 EGLImage target。

## 2.6 多线程
EGL和client API必须线程安全，EGL保障client API的命令流的顺序，但不能保障渲染到同一个surface的client API和native API之间的顺序。

client api命令不保障原子性，否则，某些此类命令可能会损害用户对平台的交互式使用。例如，在没有图形硬件的系统上渲染大型纹理映射多边形，或绘制大型 OpenGL ES 顶点数组，可能会阻止用户尽快弹出可用的菜单。

同步由client控制。通过明智地使用 glFinish、vgFinish、eglWaitClient 和 eglWaitNative 等命令以及native渲染 API 中存在的同步命令（如果存在），可以以适中的成本维护它。只要client不通过显式同步调用来阻止client API 和native渲染，就可以并行完成。

如果在client API 和native渲染之间进行不必要的切换，可能会出现一些性能下降。


# 3. EGL 函数和错误
## 3.1 错误
当前线程最近的EGL函数调用的成功或失败的信息，可以调用eglGetError获取。

返回码：
* EGL_SUCCESS：函数执行成功
* EGL_NOT_INITIALIZED：EGL未初始化或者对于特定display不能初始化。任何命令可能产生此错误。
* EGL_BAD_ACCESS：EGL不能访问该资源，比如context绑定到另一个线程。任何访问命名资源的命令可能产生此错误。
* EGL_BAD_ALLOC：分配资源失败。任何配置资源的命令可能产生此错误。
* EGL_BAD_ATTRIBUTE：未知属性或属性值被传到attribute list，任何传递属性参数或属性列表的命令可能产生此错误。
* EGL_BAD_CONTEXT：传入的EGLContext参数不是有效（valid）的。
* EGL_BAD_CONFIG：传入的EGLConfig参数不是有效的。
* EGL_BAD_CURRENT_SURFACE：当前surface不再有效。
* EGL_BAD_DISPLAY：传入的EGLDisplay参数不是有效的。
* EGL_BAD_SURFACE：传入的用于渲染的EGLSurface参数不是有效的。
* EGL_BAD_MATCH：参数不一致，比如有效的上下文需要的buffer不是由有效的surface分配。
* EGL_BAD_PARAMETER：参数无效。
* EGL_BAD_NATIVE_PIXMAP：EGLNativePixmapType参数没有对应有效的native pixmap。
* EGL_BAD_NATIVE_WINDOW：EGLNativeWindowType参数没有对应有效的native window。
* EGL_CONTEXT_LOST： power management（能耗管理）事件产生。应用必须销毁所有context并重新初始化client api状态和对象。

eglGetError是当前线程的第一个EGL调用或者在eglReleaseThread调用之后立即调用，会返回EGL_SUCCESS。

## 3.2 初始化
通过 `eglGetPlatformDisplay`（1.5版本引入） 可以得到 EGLDisplay，返回的native平台的EGLDisplay由platform参数指定。相同参数多次调用 eglGetPlatformDisplay 会返回同一个EGLDisplay handle。

通过 `eglGetDisplay`（1.0引入） 也可以得到 EGLDisplay，相比 `eglGetPlatformDisplay`更适用于不需要特定平台扩展的场景
