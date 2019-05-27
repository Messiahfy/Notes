## 1.概述
&emsp;&emsp;圆形揭露动画在`andoird 5.0`引入，圆形揭露动画为视图提供连贯的显示或隐藏动画，使用`ViewAnimationUtils.createCircularReveal()`方法。
![显示时以左上角为圆心，隐藏时以中心为圆心](https://upload-images.jianshu.io/upload_images/3468445-d00f4d7b924874b5.gif?imageMogr2/auto-orient/strip)
## 2.使用
使用`ViewAnimationUtils`类的静态方法`createCircularReveal(View view, int centerX,  int centerY, float startRadius, float endRadius)`来创建圆形揭露动画。
```
public final class ViewAnimationUtils {
    private ViewAnimationUtils() {}
   
     * @param view 执行圆形揭露动画的View
     * @param centerX 圆形动画的圆心X坐标，相对于执行动画的View
     * @param centerY 圆形动画的圆心Y坐标，相对于执行动画的View
     * @param startRadius 圆形动画的开始半径
     * @param endRadius 圆形动画的结束半径
     */
    public static Animator createCircularReveal(View view,
            int centerX,  int centerY, float startRadius, float endRadius) {
        return new RevealAnimator(view, centerX, centerY, startRadius, endRadius);
    }
}
```
## 3.开始的例子对应代码
```
button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (viewGroup.getVisibility() != View.VISIBLE) {
                    //显示，以view的左上角为圆心
                    //勾股定理函数得到对角线为结束半径
                    float finalRadius = (float) Math.hypot(viewGroup.getWidth(), viewGroup.getHeight());
                    //以左上角（x:0,y:0），初始半径0，结束半径为对角线
                    Animator animator = ViewAnimationUtils.createCircularReveal(viewGroup, 0, 0, 0, finalRadius);
                    //为什么显示时，不再动画监听器的onAnimationEnd中设VISIBLE
                    //因为隐藏就看不到动画效果了
                    viewGroup.setVisibility(View.VISIBLE);
                    animator.start();
                } else {
                    //隐藏，以view的中心为圆心
                    //获得view的中心点坐标
                    int cx = viewGroup.getWidth() / 2;
                    int cy = viewGroup.getHeight() / 2;
                    //中心点到顶点的对角线长度为初始半径
                    float initialRadius = (float) Math.hypot(cx, cy);
                    //以中心为圆心，中心点到顶点的对角线为初始半径
                    Animator animator = ViewAnimationUtils.createCircularReveal(viewGroup, cx, cy, initialRadius, 0);
                    animator.addListener(new AnimatorListenerAdapter() {
                        @Override
                        public void onAnimationEnd(Animator animation) {
                            viewGroup.setVisibility(View.INVISIBLE);
                        }
                    });
                    animator.start();
                }
            }
        });
```