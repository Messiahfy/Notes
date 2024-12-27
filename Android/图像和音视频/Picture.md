记录Canvas的绘制操作，下次直接使用picture绘制

```
// 1.创建Picture
private Picture mPicture = new Picture();

---------------------------------------------------------------

// 2.录制内容方法
private void recording() {
    // 开始录制 (接收返回值Canvas)
    Canvas canvas = mPicture.beginRecording(500, 500);
    // 创建一个画笔
    Paint paint = new Paint();
    paint.setColor(Color.BLUE);
    paint.setStyle(Paint.Style.FILL);

    // 在Canvas中具体操作
    // 位移
    canvas.translate(250,250);
    // 绘制一个圆
    canvas.drawCircle(0,0,100,paint);

    mPicture.endRecording();
}

---------------------------------------------------------------

// 3.在使用前调用(我在构造函数中调用了)
  public Canvas3(Context context, AttributeSet attrs) {
    super(context, attrs);
    
    recording();    // 调用录制
}
```
录制好后，使用方式如下：
| 序号 | 简介 |
| -- | -- |
| 1 |使用Picture提供的draw方法绘制。|
| 2 |使用Canvas提供的drawPicture方法绘制。|
| 3 |将Picture包装成为PictureDrawable，使用PictureDrawable的draw方法绘制。|