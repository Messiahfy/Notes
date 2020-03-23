## 概述
Android中用Bitmap表示一张加载到内存的图片，Bitmap的大小取决于分辨率和颜色模式，而这个分辨率不一定是原图的分辨率，例如可以根据加载到的View调整，从而节省内存。而图片文件的大小和加载到内存中的大小是不同的，图片文件的格式是图片的容器，例如jpg、png、webp等，它们把图片原本的像素信息通过特定的压缩算法转换为另一种数据格式，加载图片文件到内存，会解析图片格式，还原为位图，即Bitmap对象。

Bitmap如果是从res中的drawable目录加载的，也就是通过decodeResource方法，那么此方法内会根据设备实际dpi和drawable目录的dpi的倍数关系来方法或缩小Bitmap的分辨率，而如果没有做修改，那么同一个原图文件加载的Bitmap大小是一样的。所以，实质上如果没有手动修改了Bitmap的分辨率和颜色模式，那么原图加载为Bitmap占用内存的大小，不会受View的宽高等因素影响。

关于位图的内存管理，可参考官方文档 https://developer.android.google.cn/topic/performance/graphics

canvas(bitmap)可以使用canvas在此bitmap上绘制，也就是绘制一个bitmap

## 一.Bitmap的大小
`Bitmap`自身方法主要是get和set的属性方法，例如宽高、字节大小。还有compress方法用于压缩。

&emsp;&emsp;&#160;Bitmap有两个内部枚举类，如下：
1. `Config`：表示bitmap每个像素的颜色模式，分为`ALPHA_8`，`RGB_565`，`ARGB_8888`等
2. `CompressFormat`：表示bitmap的压缩格式，分为`JPEG`,`PNG`,`WEBP`
config相关方法有`getConfig` 和 `setConfig`（实际调用reconfigure），修改方法不应用于正在被视图系统、canvas或者Android Bitmap NDK API使用的bitmap，参见官网。

CompressFormat用于调用bitmap的compress方法时传入，用于指定压缩格式。
***
一个颜色模式为ARGB_8888（一个像素占用4个字节），像素为96*96的图片，则占用内存大小为96*96*4=36864字节=36kB

## 二.Bitmap的加载
&emsp;&emsp;Bitmap有一些CreateBitmap方法，基本是通过已有的bitmap对象来创建新的对象，可用于创建一模一样的对象，也可以创建裁剪、旋转等操作后的对象。
&emsp;&emsp;一般会使用BitmapFactory类中的四类方法来创建bitmap对象
1. `decodeByteArray`   从字节数组中加载
2. `decodeFile`   从文件中加载
3. `decodeResource`   从资源中加载，R.drawable或者R.mipmap等
4. `decodeStream`   从输入流中加载

&emsp;&emsp;四种加载方式都有一个可以传入`BitmapFactory.Options`对象的重载方法，下面了解`BitmapFactory.Options`
## 三.BitmapFactory.Options
&emsp;&emsp;BitmapFactory.Options包含以下字段（忽略deprecated ）：
* `inBitmap`  Bitmap对象，用于加载新的bitmap时，重用inBitmap的内存，被复用的Bitmap的内存大小必须大于等于被加载的Bitmap的内存大小
* `inDensity` int， 给加载的bitmap设置像素密度,如果inScaled为true，会缩放至inTargetDensity
* `inJustDecodeBounds`  boolean ，如果设置为true，不会将bitmap加载到内存，只是获取bitmap对象的信息，如outHeight、outWidth等outXXX的值
* `inMutable`  boolean，inBitmap在inMutable为true时才有效
* `inPreferredConfig`  Bitmap.Config，Bitmap对象颜色配置信息，默认是Bitmap.Config.ARGB_8888。
* `inPremultiplied`  boolean，如果为true（默认为true），返回的bitmap的颜色通道会预乘透明通道
* `inSampleSize`  int，缩小倍数，如果是4，则返回的bitmap为原来大小的1/4，<1按1处理，且任何值都会向下舍至2的倍数来处理
* `inScaled`  boolean，是否可以被缩放，如果设置，且inDensity和inTargetDensity都不为0，就会缩放至inTargetDensity。
* `inScreenDensity`  int，实际的屏幕像素密度，这纯粹适用于在密度兼容性代码中运行的应用程序
* `inTargetDensity`  最终像素密度，和inDensity、inScaled配合
* `inTempStorage`  byte[]，编码的暂时存储
* `outColorSpace`  ColorSpace，色域
* `outConfig`  加载的bitmap的颜色模式
* `outHeight`  加载的bitmap的高度
* `outMimeType`  加载的图片的mimetype，如image/png，image/png
* `outWidth`  加载的bitmap的宽度

## 四.BitmapRegionDecoder
加载部分区域，可用于加载大图，滑动显示部分区域。可参考[鸿洋高清加载巨图方案](https://blog.csdn.net/lmj623565791/article/details/49300989/)

## 五.加载bitmap，防止内存过大
&emsp;&emsp;[官网加载图片方式](https://developer.android.google.cn/topic/performance/graphics/load-bitmap)
&emsp;&emsp;很多情况下图片的大小比显示区域大，会造成额外的内存开销，所以应该加载较小的子采样版本。
### 5.1 读取Bitmap尺寸和类型
&emsp;&emsp;`BitmapFactory`的`decodeXXX`系列方法都会为构造的bitmap`分配内存`，因此容易导致`OutOfMemory`异常。每个decodeXXX方法都有一个可以传入BitmapFactory.Options的重载版本，设置`inJustDecodeBounds`为`true`可以避免内存分配，返回bitmap为`null`但是设置了`outWidth`、`outHeight`、`outMimeType`，此技术可以在构造（和内存分配）Bitmap之前读取图片的尺寸和类型。
```
BitmapFactory.Options options = new BitmapFactory.Options();
options.inJustDecodeBounds = true;
BitmapFactory.decodeResource(getResources(), R.id.myimage, options);
int imageHeight = options.outHeight;
int imageWidth = options.outWidth;
String imageType = options.outMimeType;
```
得到了Bitmap的尺寸和类型，可以在加载到内存前检查大小
### 5.2 将缩小版本加载到内存中
&emsp;&emsp;先计算采样大小：
```
public static int calculateInSampleSize(
            BitmapFactory.Options options, int reqWidth, int reqHeight) {
    // 图片的原始宽高
    final int height = options.outHeight;
    final int width = options.outWidth;
    int inSampleSize = 1;

    if (height > reqHeight || width > reqWidth) {

        final int halfHeight = height / 2;
        final int halfWidth = width / 2;

        // 计算最大的inSampleSize值，该值为2的幂，并保持高度和宽度都不小于请求的高
        // 度和宽度。
        while ((halfHeight / inSampleSize) >= reqHeight
                && (halfWidth / inSampleSize) >= reqWidth) {
            inSampleSize *= 2;
        }
    }

    return inSampleSize;
}
```
&emsp;&emsp;要使用上述方法，首先要将`inJustDecodeBounds`设为true，传递`options`参数调用方法，然后使用得到的新的`inSampleSize`值并且将`inJustDecodeBounds`设为false重新decodeXXX：
```
public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId,
        int reqWidth, int reqHeight) {

    // 第一次解码，设置 inJustDecodeBounds=true 来检查尺寸
    final BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeResource(res, resId, options);//此方法调用后，options.outHeight等值已被设置

    // 计算 inSampleSize
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);

    // Decode bitmap with inSampleSize set
    options.inJustDecodeBounds = false;
    return BitmapFactory.decodeResource(res, resId, options);
}
```
使用举例：
```
mImageView.setImageBitmap(
    decodeSampledBitmapFromResource(getResources(), R.id.myimage, 100, 100));
```