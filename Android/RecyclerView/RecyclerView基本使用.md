`RecyclerView`**最基本**的使用只需要三步：
1. 创建`RecyclerView`实例
2. `setLayoutManager()`设置布局管理器
3. `setAdapter()`设置适配器

在基本的使用中只需要自定义`adapter`和`viewHolder`。
## 1. 自定义adapter和viewHolder
自定义adapter至少要重写三个方法，如下：
```
public class HfyAdapter extends RecyclerView.Adapter<HfyViewHolder> {
    private ArrayList<String> data;

    public HfyAdapter(ArrayList<String> data) {
        this.data = data;
        //可以改变this.data指向的对象，调用notifyData...方法有效，但如果修改传进来的外部data变量，调 
        //用notifyData...方法无效，因为下面的方法中都是使用的this.data这个对象变量，只认this.data这个变量。
        //不要认为内部记录了this.data引用的对象本身。只是在绑定数据时使用this.data这个对象变量，使用它引用的数据。
        //内部只是缓存ViewHolder，并没有缓存数据
    }

    @NonNull
    @Override
    public HfyViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        return new HfyViewHolder(LayoutInflater.from(parent.getContext()).inflate(R.layout.item, parent, false));
        //viewType参数会赋值给viewHolder.mItemViewType
    }

    @Override
    public void onBindViewHolder(@NonNull HfyViewHolder holder, int position) {
        //ViewHolder实例中有itemView这个public实例域，它就是创建ViewHolder时传入的view
        TextView textView = holder.itemView.findViewById(R.id.text_view);
        textView.setText(data.get(position));
        //或者在ViewHolder类中将TextView作为类中的实例域，则可写成如下
        //holder.textView.setText(data.get(position));
    }

    @Override
    public int getItemCount() {
        return data.size();
    }

    //用于实现多类型布局
    @Override
    public int getItemViewType(int position) {
        return 1;
    }
}
```
&emsp;&emsp;`onCreateViewHolder`用于创建`ViewHolder`对象，`onBindViewHolder`用于在绑定`viewHolder`时完成某些设置，`getItemCount`用于返回数据的数量。
&emsp;&emsp;adapter常用的方法有notify...这类更新数据的方法，还有以下关于生命周期的方法回调：
* onViewRecycled()
* onViewAttachedFromWindow()
* onViewDetachedFromWindow()
* onAttachedToRecyclerView()
* onDetachedFromRecyclerView()
&emsp;&emsp;下面两个方法可以注册监听 notify...() 系列方法的事件，当调用了 notify...() 系列的方法时，注册监听后就可以接收到回调。
* registerAdapterDataObserver()
* unregisterAdapterDataObserver()

&emsp;&emsp;而最简单的自定义的ViewHolder如下：
```
public class HfyViewHolder extends RecyclerView.ViewHolder {
    //TextView textView;
    HfyViewHolder(View itemView) {
        super(itemView);//关键就是传入每一项要显示的view
        //textView = itemView.findViewById(R.id.xxxx);  如前面所说，可以进行一些子视图的设置
    }
}
```
&emsp;&emsp;关于ViewHolder比较关键的是public域`itemView`和getAdapterPosition ()、getLayoutPosition ()、getItemViewType()

getAdapterPosition ()和getLayoutPosition ()的差别在于getAdapterPosition ()是数据源中的实时位置，getLayoutPosition ()是显示在界面的位置，一般情况是相同的，只有在更新数据源但还没有刷新界面的短暂时间内是不相等的。

getItemViewType()用于获得设置的viewType

> 1.2.0版本中，增加`MergeAdapter`实现多类型布局，但动态化程度并不是很强，只能按组合adapter的顺序来显示不同的布局。比如按顺序添加header、content、footer对应的adapter；更动态化的布局，还是要使用`getItemViewType`

## 2. ItemDecoration
&emsp;&emsp;可以自定义`ItemDecoration`来添加分割线或其他对列表中的项目修饰。通过recyclerView.addItemDecoration (RecyclerView.ItemDecoration decor)方法。
&emsp;&emsp;ItemDecoration是一个抽象类，需要我们自己实现。去除掉onDraw和onDrawOver的`@Deprecated`的方法，代码如下：
```
public abstract static class ItemDecoration {
        /**
         * 提供给RecyclerView的Canvas中绘制任何适当的装饰。此方法绘制的任何内容 
         * 都将在绘制项目视图之前绘制，因此将显示在视图下方（Z轴）。
         *
         * @param c Canvas to draw into
         * @param parent RecyclerView this ItemDecoration is drawing into
         * @param state The current state of RecyclerView
         */
        public void onDraw(Canvas c, RecyclerView parent, State state) {
            onDraw(c, parent);//此Deprecated方法也为空
        }

        /**
         * 提供给RecyclerView的Canvas中绘制任何适当的装饰。此方法绘制的任何内容 
         * 都将在绘制项目视图之后绘制，因此将显示在视图上方（Z轴）。
         */
        public void onDrawOver(Canvas c, RecyclerView parent, State state) {
            onDrawOver(c, parent);//此Deprecated方法也为空
        }

        @Deprecated
        public void getItemOffsets(Rect outRect, int itemPosition, RecyclerView parent) {
            outRect.set(0, 0, 0, 0);
        }

        /**
         * 设置给定item的任何偏移量。outRect的每个字段都指定item视图应插入的像素 
         * 数，类似于padding或margin。默认实现将outRect的边界设置为0并返回。
         *
         * 如果此ItemDecoration不影响item视图的定位，则应在返回之前将outRect（left， 
         * top，right，bottom）的所有四个字段设置为零。
         *
         * If you need to access Adapter for additional data, you can call
         * RecyclerView#getChildAdapterPosition(View) to get the adapter position of the
         * View.
         *
         * @param outRect Rect to receive the output.
         * @param view    The child view to decorate
         * @param parent  RecyclerView this ItemDecoration is decorating
         * @param state   The current state of RecyclerView.
         */
        public void getItemOffsets(Rect outRect, View view, RecyclerView parent, State state) {
            getItemOffsets(outRect, ((LayoutParams) view.getLayoutParams()).getViewLayoutPosition(),
                    parent);
        }
    }
```
#### 2.1 getItemOffsets()
&emsp;&emsp;对outRect设置四个方向的偏移量，根据item数量会被循环调用从而实现对每个item view实现类似margin的效果。可以通过view参数获取位置等数据用于过滤等操作。
```
@Override
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
        outRect.set(30, 10, 30, 10);//将在每个item的四个方向产生偏移
    }
```
将recyclerView的background设为绿色，产生偏移后，偏移的地方露出了背景色，效果如下：
![效果图](http://upload-images.jianshu.io/upload_images/3468445-846b3f61a9eba460.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1080/q/50)
#### 2.2 onDraw()
&emsp;&emsp;`onDraw()`方法绘制在item的下面（z轴），所以需要和`getItemOffsets()`方法配合使用，利用`getItemOffsets()`撑开间隔区域，在间隔区域绘制，才不会被`item`遮住（除非`item`背景透明）。
&emsp;&emsp;需要注意的是，`getItemOffsets()` 是针对每一个 `ItemView`，而 `onDraw` 方法却是针对 `RecyclerView` 本身，所以在 `onDraw` 方法中需要遍历屏幕上可见的 `ItemView`，分别获取它们的位置信息，然后分别的绘制对应的分割线。
```
public class HfyDecoration extends RecyclerView.ItemDecoration {
    private Paint paint;

    public HfyDecoration() {
        paint = new Paint();
        paint.setColor(Color.RED);
        paint.setStyle(Paint.Style.FILL);
        paint.setAntiAlias(true);
    }

    @Override
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
        //第一个ItemView不需要在上面绘制分割线
        if (parent.getChildAdapterPosition(view) != 0) {
            outRect.set(0, 10, 0, 0);
        }
    }

    @Override
    public void onDraw(Canvas c, RecyclerView parent, RecyclerView.State state) {
        super.onDraw(c, parent, state);
        int childCount = parent.getChildCount();
        for (int i = 0; i < childCount; i++) {
            View view = parent.getChildAt(i);
            int index = parent.getChildAdapterPosition(view);
            //第一个ItemView不需要绘制
            if (index == 0) {
                continue;
            }
            float left = view.getLeft();
            float top = view.getTop() - 10;
            float bottom = view.getTop();
            float right = view.getRight();
            //绘制矩形作为分割线
            c.drawRect(left, top, right, bottom, paint);
        }
    }
}
```
效果如下:
![分割线](http://upload-images.jianshu.io/upload_images/3468445-3a6a8775fea639fc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1080/q/50)
&emsp;&emsp;也可以使用canvas绘制其他各种图形，或者部分被item遮住的情况，利用getItemOffsets来设置间隔，以达到各种效果。
#### 2.3 onDrawOver()
`onDrawOver()`除了是绘制在item之上（z轴），其他用法和`onDraw()`一致。可与前两者配合实现联系人首字母粘性header等效果。
#### 2.4 注意点
&emsp;&emsp;`ItemDecoration` 的 `onDraw()`在`RecyclerView`的`onDraw()`中被调用，RecyclerView的onDraw()在RecyclerView父类View的draw()（RecyclerView的draw()调用了super.draw()）中被调用，onDrawOver()在RecyclerView的draw()中调用，即onDraw()和onDrawOver()都会在Draw()中被调用，所以这两个方法是会被滑动触发的，滑动的时候会不断调用，方法中获取的数据是实时的。
&emsp;&emsp;而`getItemOffsets()`在RecyclerView初次布局子item时就会对每个子item都会调用，次数为显示出来的子item个数；而在滑动时只有显示出新的item时候才会调用，滑动而没有滑动到显示出新的item时不会调用。由于RecyclerView的复用机制，所以此处的**新的item**指未显示到显示的情况，即使复用，比如一个item显示会调用`getItemOffsets()`，它滑动出屏幕后又滑动进来，就会再次调用。

### 3.LayoutManager
布局管理器用于列表的所有item排列布局，系统提供`LinearLayoutManager`，`GridLayoutManager`和`StaggeredGridLayoutManager`三种排列方式，也可以自定义。
一些方法说明：
* findFirstCompletelyVisibleItemPosition()  获得第一个完全可见的item的位置
* findFirstVisibleItemPosition() 获得第一个可见的item的位置
* findLastCompletelyVisibleItemPosition() 获得最后一个完全可见的item的位置
* findLastVisibleItemPosition() 获得最后一个可见的item的位置
完全可见的判断和水平还是垂直有关，如果是水平方向，则左右完全可见即是完全可见，垂直同理。

### 4.ItemAnimator
用于列表item的增加、删除、移动的动画，ItemAnimator是一个抽象类，可以自定义，系统也提供了默认的实现类。

### 5.ItemTouchHelper
用于支持侧滑删除和拖拽的辅助类

### 6.RecyclerView的OnScrollListener和OnFlingListener
设置关于滑动和fling（滑动松手到停止滑动的操作）的操作。
OnFlingListener有一些实现的子类：抽象类SnapHelper，以及其子类PagerSnapHelper、LinearSnapHelper、GravitySnapHelper，可以实现滑动松手后到滑动停止的具体效果，例如类似ViewPager翻页，或者将Item对齐到某个位置等效果，而不是停在任意位置。

### 7.局部刷新
notifyItemChanged(position, payload)