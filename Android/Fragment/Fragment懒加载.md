管理fragment一般有add+hide方式和ViewPager方式这两种。
## 1.显示和隐藏的相关细节
1. `show()`和`hide()`方法不会影响生命周期，只会回调`onHiddenChanged(boolean hidden)`方法，`isHidden()`的返回值也只和这两个方法有关，默认`false`。
2. 锁屏、home等操作会影响生命周期，但不会影响`isHidden()`和`isVisible()`的返回值。
3. `add`两个`fragment`，而没有`remove`或者`hide`前一个，则后一个实际不会显示，但`isVisible()`会返回`true`，可查看`isVisible()`的源码即知影响其返回值的因素。
4. `setUserVisibleHint()`方法一般并没有调用，主要是在`ViewPager`中被使用。它并不会使`fragment`显示或隐藏，它的关键只是在于`hint`（提示），提示当前`fragment`是否显示。例如在`ViewPager`中，显示`fragment1`时，预加载相邻的`fragment2`虽然看不到但也是`isVisible()==true`和`isHidden()==false`的，可看作**显示在屏幕范围外**。ViewPager就对fragment1和fragment2分别调用`setUserVisibleHint(true)`和`setUserVisibleHint(false)`，可以调用`getUserVisibleHint()`方法来得知某个fragment是否**可见**（用户能看到）。`ViewPager`中调用 `setUserVisibleHint()`在`fragment`的`onAttach()`之前。所以我们可以通过`setUserVisibleHint()`和`getUserVisibleHint()`实现比`isHidden()`和`isVisible()`更准确的判断是否**可见**的效果。

## 2.Fragment在ViewPager中的生命周期
&emsp;&emsp;`ViewPager`会默认预加载当前显示的`fragment`**左右相邻**的`fragment`，可以通过`viewPager.setOffscreenPageLimit()`方法修改左右相邻预加载fragment的个数，如果设为2，就会预加载当前显示的fragment左右分别两个fragment，但只能最少设为1。
&emsp;&emsp;1.以`FragmentPagerAdapter`为例，在ViewPager中使用三个fragment，OffscreenPageLimit使用默认左右分别1个。初始显示，生命周期如下：
```
1setUserVisibleHint: false
2setUserVisibleHint: false
1setUserVisibleHint: true
1onAttach: 
1onCreate: 
2onAttach: 
2onCreate: 
1onCreateView: 
1onActivityCreated: 
1onStart: 
1onResume: 
2onCreateView:
2onActivityCreated: 
2onStart: 
2onResume: 
```
&emsp;&emsp;可以看到`fragment1`和`fragment2`的生命周期都走到了`onResume()`，不同的是，`ViewPager`调用了当前显示的`fragment1`的`setUserVisibleHint(true)`。
***********************************
&emsp;&emsp;2.然后滑动到显示`fragment2`，生命周期变化如下：
```
3setUserVisibleHint: false
1setUserVisibleHint: false
2setUserVisibleHint: true
3onAttach: 
3onCreate: 
3onCreateView: 
3onActivityCreated: 
3onStart: 
3onResume: 
```
&emsp;&emsp;显示`fragment2`的时候，加载了`fragment3`，并且其生命周期走到了`onResume()`，验证了上面所说会默认预加载左右分别1个fragment。此时`ViewPager`调用了`fragment2`的`setUserVisibleHint(true)`，而设1和3为false。
***********************************
&emsp;&emsp;3.接着滑动到显示`fragment3`，生命周期变化如下：
```
2setUserVisibleHint: false
3setUserVisibleHint: true
1onPause: 
1onStop: 
1onDestroyView: 
```
&emsp;&emsp;显示`fragment3`的时候，`fragment1`并不是3的左右相邻一位的fragment，所以`fragment1`的生命周期走到了`onDestroyView()`，销毁了其视图结构，但并未销毁对象。此时3 `setUserVisibleHint`为true，2为false。
**************************************
&emsp;&emsp;4.如果直接从1跳转3,则3并没有经过预加载
```
3setUserVisibleHint: false
1setUserVisibleHint: false
3setUserVisibleHint: true
3onAttach: 
3onCreate: 
3onCreateView:
3onActivityCreated: 
3onStart: 
3onResume: 
1onPause: 
1onStop: 
1onDestroyView: 
```
**************************************
**注意：**以上生命周期测试使用的是`FragmentPagerAdapter`，还有一个`FragmentStatePagerAdapter`用于fragment较多的情况。区别在于如果某个fragment不在当前显示的fragment的**预加载相邻范围内**，`FragmentPagerAdapter`对该fragment调用**detach()**方法，虽然此函数名为detach，但只会使fragment的生命周期走到`onDestroyView()`，并不会调用`onDestroy()`和`onDetach()`，而`FragmentStatePagerAdapter`则会调用`remove()`，使fragment走到`onDetach()`。即`FragmentPagerAdapter`只会销毁视图结构，而`FragmentStatePagerAdapter`还会销毁fragment对象。
**************************************
## 3.在ViewPager中懒加载
> 因为`fragment`在`ViewPager`中使用时才会被预加载，所以我们说的`fragment`懒加载都是针对其在`ViewPager`使用的情况

&emsp;&emsp;通过以上生命周期看到，在ViewPager中通过onResume()并不能判断是否对用户**可见**，而上面也说了isHidden()和isVisible()也不能判断。所以只能通过重写`setUserVisibleHint()`和调用`getUserVisibleHint()`来得知是否实际对用户可见。
&emsp;&emsp;`ViewPager`中的**懒加载**就是要在fragment实际对用户可见时才执行网络请求等任务，所以就要利用`setUserVisibleHint()`方法来实现。
&emsp;&emsp;`setUserVisibleHint()`是被`FragmentPagerAdapter`主动调用的，一是在adapter的`instantiateItem`方法中调用，**实例化**fragment时设为`false`，二是在`setPrimaryItem`方法中把**将要显示**的fragment设为`true`，**上一个显示**的fragment设为`false`。
&emsp;&emsp;**要注意的是：`setUserVisibleHint()`的初次调用是在fragment的`onAttach()`回调之前，且都是设为false。而第一个显示的fragment或者从1直接跳转3这种情况会在`onAttach()`回调前第二次调用并设为true，因为这种情况fragment没有预加载；预加载的fragment的`setUserVisibleHint()`设为true会在实际可见的时候才会调用。**所以`setUserVisibleHint(true)`时fragment的视图并不一定创建完成。

## 4.嵌套ViewPager中的懒加载
![网易云音乐为例](https://upload-images.jianshu.io/upload_images/3468445-2c31fa0d2a8d670e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
ParentFragment1显示时，会预加载ParentFragment2，ParentFragment2会走到生命周期onResume，而且会把其中的childFragment1加载出来并setUserVisibleHint true，childFragment2也被预加载出来，但childFragment2和未嵌套一样是setUserVisibleHint false。所以这里的关键就是childFragment1并不可见的情况下加载出来并setUserVisibleHint true，这种情况就要比未嵌套多考虑一下父fragment的UserVisibleHint状态，因为这是虽然childFragment1并不可见却setUserVisibleHint true，但其父fragment也就是ParentFragment2是setUserVisibleHint false，可以根据这个信息来处理。

仅考虑懒加载（也就是第一次实际可见）较简单，但是要考虑嵌套情况的可见与不可见，还需考虑更多情况。
对于可见状态的生命周期调用顺序，父 Fragment总是优先于子 Fragment，而对于不可见事件，内部的 Fragment 生命周期总是先于外层 Fragment。（网友所言，待测试）

> 懒加载实质上就是判断第一次对用户实质可见的情况，根据业务情况还可以根据以上所有分析来得出切换界面的每次可见和不可见的状态