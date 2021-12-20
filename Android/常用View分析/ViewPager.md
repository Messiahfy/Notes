## ViewPager必须使用Adapter
如果不使用Adapter，而是通过xml或者代码中添加子View，宽高均会为0，所以**必须配合Adapter使用**。原因如下：
1. 先尝试xml添加子View：ViewPager中测量时会使用widthFactor来计算child的宽度，由于widthFactor默认为0，并且没有提供xml中widthFactor对应的自定义属性，无法在xml中设置widthFactor，那么xml中添加的子view宽度都会为0。而在onLayout中，会对`infoForChild`方法返回不为null的child才执行布局过程，`infoForChild`方法中会对`mItems`属性与child通过`Adapter.isViewFromObject`来判断，但如果没有设置Adapter，`mItems`的长度会为0，也就不会执行child的layout，所以child的宽高都会为0。
2. 如果在代码中添加子View：LayoutParams内的widthFactor等属性，是包内可见，无法在外部使用。同上，即使可以使用，因为没有设置Adapter，也不会执行布局流程。

## child的宽高
ViewPager的LayoutParams的无参构造函数，宽高会默认设置为MATCH_PARENT

## Adapter的使用
这里介绍几个主要的需要重写的方法，还有一些不常用的方法可以看源码了解用法。
* instantiateItem：在ViewPager的populate方法中调用，也就是把返回的Object包装到`ItemInfo`中，并添加到`mItems`列表，会在当前位置对应的`ItemInfo`不存在的时候调用。因为只是返回一个Object，并没有addView，所以需要我们自己调用addView。
```
public Object instantiateItem(@NonNull ViewGroup container, int position)
```
* getCount：页面数量。
```
public abstract int getCount()
```
* destroyItem：ViewPager的页面有缓存大小，默认缓存当前左右各1个页面，如果不在缓存范围内，则会调用此方法，我们就需要在这里调用removeView。

```
public void destroyItem(@NonNull ViewGroup container, int position, @NonNull Object object)
```
* isViewFromObject：ViewPager的infoForChild方法中，就是通过这个方法，来确定child view和`instantiateItem`返回的object的对应关系，从而通过child view得到`mItems`列表中对应的`ItemInfo`。
```
public abstract boolean isViewFromObject(@NonNull View view, @NonNull Object object)
```

## 滚动方式
同样是使用scrollTo

## PageTransformer
设置PageTransformer可以在页面滚动过程中，设置一些动画效果。
```
public interface PageTransformer {
    // 这个方法在滚动的时候会不断调用。position是相对于当前中间view的位置，比如当前位置的左边为-1，当前为0，右边为1。。
    void transformPage(@NonNull View page, float position);
}
```

## FragmentPagerAdapter和FragmentStatePagerAdapter
FragmentPagerAdapter中在调用`destroyItem`时，会对Fragment调用FragmentTransaction的detach方法，这会使Fragment调用到onDestroyView()，但不会执行onDestroy()和onDetach()，也就是Fragment会销毁视图，但Fragment本身的实例还会持续存在。所以FragmentPagerAdapter更适合数量较小的情况，并且在缓存范围外的Fragment由于主动销毁了视图，所以也不会保存视图状态。

FragmentStatePagerAdapter在调用`destroyItem`时，会对Fragment调用FragmentTransaction的remove方法，这既会销毁视图，也会销毁Fragment本身的实例，所以更适合Fragment数量较多的情况，可以减少内存占用。虽然会主动销毁视图和Fragment实例，但是会通过`Fragment.SavedState`列表来保存每个Fragment的状态，可以在进入缓存范围内时，创建Fragment，并给它设置之前保存的状态；同时，这个状态也会在低内存时应用进程被回收和恢复的情况，通过Adapter的`saveState`和`restoreState`来保存了Fragment的状态。

总结来说，FragmentPagerAdapter适合数量较小的情况，虽然会保存缓存范围外的Fragment的实例，但并且不会保存对应的视图和状态；而FragmentStatePagerAdapter适合数量较大的情况，虽然不会保存缓存范围外的Fragment的实例，但是会保存状态。