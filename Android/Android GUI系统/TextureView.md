TextureView由Android 4.0引入，是一个结合了View特性和SurfaceTexture的View对象，它和普通的View一样是通过 UI Renderer 绘制的，因此和View完全兼容，不会有SurfaceView无法做动画，无法平移、放缩等种种问题问题。

SurfaceTexture是TextureView中的一个核心组件。
纹理（Texture）是OpenGL中的一个概念。切确地说，Texture是OpenGL中的一种类型，通常存放着一张或多张图像数据，在渲染中使用

SurfaceTexture从命名上来看，是一种由Surface转换过来的GL Texture，它可以将 Surface 提供的图像流转换为 GL 的纹理，然后对该内容进行二次处理，最终被处理的内容可以被交给其他消费者消费。

3.1 SurfaceView比TextureView好的地方
- TextureView 需要通过 UI Renderer 输出，性能会比在独立线程输出的SurfaceView 会有明显下降（低端机，高 GPU 负荷场景可能存在 15% 左右的帧率下降）
- SurfaceTexture内置的BufferQueue会导致它比SurfaceView消耗更多的内存
- 因为需要在SurfaceTexture自己的线程，MainThread，RenderThread三个线程之间进行写读同步，当同步失调的时候，比较容易出现掉帧或者吞帧导致的卡顿和抖动现象。TextureView会有1~3帧的延迟（三缓冲机制）。
- 由于SurfaceTexture的特性，TextureView需依赖硬件加速
- SurfaceView的Surface完全由系统管理，不用担心生命周期和性能等问题
3.2 TextureView比SurfaceView好的地方
- 可当做一个View使用进行各种动画和平移放缩操作，完全兼容View
- SurfaceView的Surface完全由系统管理也是一个缺点，无法进行如离屏渲染之类的操作
