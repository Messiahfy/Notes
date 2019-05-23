ViewOverlay有add(Drawable drawable)、remove(Drawable drawable)和clear()方法

ViewGrouplay继承自ViewOverlay，所以继承来的上述方法，还有add(View view)、remove(View view)

添加的view是不能接收任何事件的，相当于drawable，只是可以看。
添加的view可以移动到viewGroup之外，相当于浮在上一层的效果，一般可以用于动画