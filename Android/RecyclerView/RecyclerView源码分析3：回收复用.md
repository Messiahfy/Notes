文章记录：https://juejin.cn/post/6984974879296585764

RecyclerView的视图缓存分为四层，分别是mAttachedScrap、mCachedViews、mViewCacheExtension（一般不用）、mRecyclerPool

1.**mAttachedScrap**：屏幕内 item 快速复用（RecyclerView具有两次 onLayout() 过程，第二次 onLayout() 中直接使用第一次 onLayout() 缓存的 View，而不必再创建）。

2.**mCachedViews**：
* ArrayList 类型
* 默认 size 为 2，size 可变
* 复用算法是从尾部倒序匹配 ViewHolder position 与传入的 position 是否相等，匹配成功则返回
* 为了优化上一步，下一个可能出现的 item 将会被置于尾部，下滑时下一个可能出现的就是底部再下一个，上滑则是顶部再上一个）。
* mCachedViews缓存屏幕外2个item（默认），适用于滑出item然后又滑进的情况（即缓存的ViewHolder的position没有改变），一般是缓存屏幕中显示的第一个item的前一个和最后一个item的下一个，不过例如滑动到最顶部时则缓存屏幕内最下面item的下面两个item。当被复用的时候是不会再走Bind流程的。

3.**mRecyclerPool** ：
* 内部维护了一个 SparseArray
* SparseArray key 为 ViewHolder 的 ViewType，这说明每一套 ViewHolder 都具有自己的缓存数据
* SparseArray value 为 ScrapData 类型，ScrapData 就是关键的缓存数据了，其数据结构简略如下：
```
static class ScrapData {
     final ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
     int mMaxScrap = 5;
     // ...
}
```
由此可见，针对每一种 ViewHolder，RecycledViewPool 都会维护一个默认大小为 5 的 ArrayList 来用做缓存它们；ArrayList 的默认大小被限制为 5，但是这个值是可以通过 RecycledViewPool#setMaxRecycledViews(viewType, max) 来修改的，比如想换成大一点的 10、20，都是可以的（这也是该数据类型为 ArrayList 而不是数组的原因之一）。

> **mCachedViews**和**mRecyclerPool** 的场景区别是：开始屏幕显示10个item，滑动到某个位置只显示了5个，**mCachedViews**只缓存邻近的2个，而在滑动过程中邻近的两个是一直在改变的，在从10个到5个的滑动过程中，会不断把滑出屏幕的放到**mCachedViews**中，**mCachedViews**满了后，每次就从**mCachedViews**取一个放到**mRecyclerPool**中，再把新回收的放到**mCachedViews**中。

![缓存对比](https://upload-images.jianshu.io/upload_images/3468445-f35548f764860d89.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 看源码前，先明确一点：滑动过程中的回收复用，是mCachedViews和mRecyclerPool 这两个缓存完成的，关于缓存机制应该重点分析这两个缓存。

#### Recycler的类结构
```
public final class Recycler {
    final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
    ArrayList<ViewHolder> mChangedScrap = null;

    final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();

    private final List<ViewHolder>
            mUnmodifiableAttachedScrap = Collections.unmodifiableList(mAttachedScrap);

    private int mRequestedCacheMax = DEFAULT_CACHE_SIZE;
    int mViewCacheMax = DEFAULT_CACHE_SIZE;

    RecycledViewPool mRecyclerPool;

    private ViewCacheExtension mViewCacheExtension;

    static final int DEFAULT_CACHE_SIZE = 2;
    ...
    类的结构也比较清楚，这里可以清楚的看到我们后面讲到的四级缓存机制所用到的类都在这里可以看到：
    * 1.一级缓存：mAttachedScrap，mChangedScrap
    * 2.二级缓存：mCacheViews
    * 3.三级缓存：mViewCacheExtension
    * 4.四级缓存：mRecyclerPool
}
```
Recycler是用于管理废弃的item或者从RecyclerView中移除的View便于复用，废弃的View是指仍然依附在RecyclerView中，但是已经被标记为移除的或者可以复用的。



## 源码分析
在布局绘制流程的分析中，`layoutChunk`方法中的`View view = layoutState.next(recycler)`用于获取`view`，如果有缓存有从缓存中获取，否则会重新创建一个。内部最终会调用到`Recycler`的`tryGetViewHolderForPositionByDeadline`方法

#### 取缓存
![自己用visio画一下](https://upload-images.jianshu.io/upload_images/3468445-a493c2a21569909c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



`Recycler`的`tryGetViewHolderForPositionByDeadline`方法获取给定位置的`ViewHolder`，要么从`scrap`、`cache`或`RecycledViewPool`获取，要么直接创建。
```
//RecyclerView.Recycler.java
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
        boolean dryRun, long deadlineNs) {
    if (position < 0 || position >= mState.getItemCount()) {
        throw //......
    }
    boolean fromScrapOrHiddenOrCache = false;
    ViewHolder holder = null;

    // 0. 首先从Attached中的Changed数组中取（有动画的情况下进入此if）

    if (mState.isPreLayout()) {
        holder = getChangedScrapViewForPosition(position);
        fromScrapOrHiddenOrCache = holder != null;
    }

    // 1. 分别从AttachedScrap,Hidden，Cached中获取ViewHolder

    if (holder == null) {
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
        if (holder != null) {
            if (!validateViewHolderForOffsetPosition(holder)) {
                // recycle holder (and unscrap if relevant) since it can't be used
                if (!dryRun) {
                    // we would like to recycle this but need to make sure it is not used by
                    // animation logic etc.
                    holder.addFlags(ViewHolder.FLAG_INVALID);
                    if (holder.isScrap()) {
                        removeDetachedView(holder.itemView, false);
                        holder.unScrap();
                    } else if (holder.wasReturnedFromScrap()) {
                        holder.clearReturnedFromScrapFlag();
                    }
                    recycleViewHolderInternal(holder);
                }
                holder = null;
            } else {
                fromScrapOrHiddenOrCache = true;
            }
        }
    }
    if (holder == null) {
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        if (offsetPosition < 0 || offsetPosition >= mAdapter.getItemCount()) {
            throw //......
        }

        final int type = mAdapter.getItemViewType(offsetPosition);

        // 2. Find from scrap/cache via stable ids, if exists
        
        if (mAdapter.hasStableIds()) {
            holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                    type, dryRun);
            if (holder != null) {
                // update position
                holder.mPosition = offsetPosition;
                fromScrapOrHiddenOrCache = true;
            }
        }

        // 3. 自定义缓存，通过mViewCacheExtension进行获取

        if (holder == null && mViewCacheExtension != null) {
            // We are NOT sending the offsetPosition because LayoutManager does not
            // know it.
            final View view = mViewCacheExtension
                    .getViewForPositionAndType(this, position, type);
            if (view != null) {
                holder = getChildViewHolder(view);
                if (holder == null) {
                    throw new IllegalArgumentException("getViewForPositionAndType returned"
                            + " a view which does not have a ViewHolder"
                            + exceptionLabel());
                } else if (holder.shouldIgnore()) {
                    throw new IllegalArgumentException("getViewForPositionAndType returned"
                            + " a view that is ignored. You must call stopIgnoring before"
                            + " returning this view." + exceptionLabel());
                }
            }
        }

        // 4. 通过mRecyclerPool获取

        if (holder == null) { // fallback to pool
            holder = getRecycledViewPool().getRecycledView(type);
            if (holder != null) {
                holder.resetInternal();
                if (FORCE_INVALIDATE_DISPLAY_LIST) {
                    invalidateDisplayListInt(holder);
                }
            }
        }

        // 创建

        if (holder == null) {
            long start = getNanoTime();
            if (deadlineNs != FOREVER_NS
                    && !mRecyclerPool.willCreateInTime(type, start, deadlineNs)) {
                // abort - we have a deadline we can't meet
                return null;
            }
            holder = mAdapter.createViewHolder(RecyclerView.this, type);
            if (ALLOW_THREAD_GAP_WORK) {
                // only bother finding nested RV if prefetching
                RecyclerView innerView = findNestedRecyclerView(holder.itemView);
                if (innerView != null) {
                    holder.mNestedRecyclerView = new WeakReference<>(innerView);
                }
            }

            long end = getNanoTime();
            mRecyclerPool.factorInCreateTime(type, end - start);
        }
    }
    //在返回给LayoutManager重新绘制前，需要更新一下ViewHolder的相关信息
    if (fromScrapOrHiddenOrCache && !mState.isPreLayout() && holder
            .hasAnyOfTheFlags(ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST)) {
        holder.setFlags(0, ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
        //记录动画信息
        if (mState.mRunSimpleAnimations) {
            int changeFlags = ItemAnimator
                    .buildAdapterChangeFlagsForAnimations(holder);
            changeFlags |= ItemAnimator.FLAG_APPEARED_IN_PRE_LAYOUT;
            final ItemHolderInfo info = mItemAnimator.recordPreLayoutInformation(mState,
                    holder, changeFlags, holder.getUnmodifiedPayloads());
            //如果当前的ViewHolder已经绑定过数据，那么记录一下动画信息
            recordAnimationInfoIfBouncedHiddenView(holder, info);
        }
    }

    boolean bound = false;
    if (mState.isPreLayout() && holder.isBound()) {
        //绑定过数据的ViewHolder
        holder.mPreLayoutPosition = position;
    } else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
        //未绑定数据的ViewHolder需要进行数据绑定
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
    }

    final ViewGroup.LayoutParams lp = holder.itemView.getLayoutParams();
    final LayoutParams rvLayoutParams;
    if (lp == null) {
        rvLayoutParams = (LayoutParams) generateDefaultLayoutParams();
        holder.itemView.setLayoutParams(rvLayoutParams);
    } else if (!checkLayoutParams(lp)) {
        rvLayoutParams = (LayoutParams) generateLayoutParams(lp);
        holder.itemView.setLayoutParams(rvLayoutParams);
    } else {
        rvLayoutParams = (LayoutParams) lp;
    }
    rvLayoutParams.mViewHolder = holder;
    rvLayoutParams.mPendingInvalidate = fromScrapOrHiddenOrCache && bound;
    return holder;
}
```

对方法中的内容具体分析：

```
if (mState.isPreLayout()) {//preLayout默认是false，只有有动画的时候才为true
    holder = getChangedScrapViewForPosition(position);
    fromScrapOrHiddenOrCache = holder != null;
}
```
有动画的时候，进入这个if，`getChangedScrapViewForPosition`方法从`mChangedScrap`中获取`holder`，因为这是动画情况下，所以没有把`mChangedScrap`看作常规缓存。

#### 第一次，尝试从mAttachedScrap和mCacheView中获取
```
if (holder == null) {
    holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
    if (holder != null) {
        if (!validateViewHolderForOffsetPosition(holder)) {
            // recycle holder (and unscrap if relevant) since it can't be used
            if (!dryRun) {
                // we would like to recycle this but need to make sure it is not used by
                // animation logic etc.
                holder.addFlags(ViewHolder.FLAG_INVALID);
                if (holder.isScrap()) {
                    removeDetachedView(holder.itemView, false);
                    holder.unScrap();
                } else if (holder.wasReturnedFromScrap()) {
                    holder.clearReturnedFromScrapFlag();
                }
                recycleViewHolderInternal(holder);
            }
            holder = null;
        } else {
            fromScrapOrHiddenOrCache = true;
         }
    }
}
```
通过`getScrapOrHiddenOrCachedHolderForPosition`方法来获取`ViewHolder`，并检验`holder`的有效性，如果无效，则从`mAttachedScrap`中移除，并加入到`mCacheViews`或者`Pool`中，并且将`holder`置`null`，走下一级缓存判断。
```
ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
    final int scrapCount = mAttachedScrap.size();

    // 先尝试从mAttachedScrap获取
    for (int i = 0; i < scrapCount; i++) {
        final ViewHolder holder = mAttachedScrap.get(i);
        if (!holder.wasReturnedFromScrap() && holder.getLayoutPosition() == position
                && !holder.isInvalid() && (mState.mInPreLayout || !holder.isRemoved())) {
            holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
            return holder;
        }
    }
    //dryRun为false
    if (!dryRun) {
         //从HiddenView中获得，这里获得是View
         View view = mChildHelper.findHiddenNonRemovedView(position);
         if (view != null) {
            // This View is good to be used. We just need to unhide, detach and move to the
            // scrap list.
            //通过View的LayoutParam获得ViewHolder
            final ViewHolder vh = getChildViewHolderInt(view);
            mChildHelper.unhide(view);//从HiddenView中移除
            int layoutIndex = mChildHelper.indexOfChild(view);
            ...
            mChildHelper.detachViewFromParent(layoutIndex);
            scrapView(view);//添加到Scrap中
            vh.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP
                    | ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
            return vh;
        }
    }

    // 从mCachedViews中获取
    final int cacheSize = mCachedViews.size();
    for (int i = 0; i < cacheSize; i++) {
        final ViewHolder holder = mCachedViews.get(i);
        // invalid view holders may be in cache if adapter has stable ids as they can be
        // retrieved via getScrapOrCachedViewForId
        if (!holder.isInvalid() && holder.getLayoutPosition() == position) {
            //可以看到，如开始所说，mCachedViews是缓存相邻滑出去的item，滑进来时position相同 
            //才认为是一个viewholder，且不用再bind
            if (!dryRun) {
                mCachedViews.remove(i);
            }
            return holder;
        }
    }
    return null;
}
```
这里分了三个步骤：

1.从mAttachedScrap中获取
2.从HiddenView中获取
3.从CacheView获取
![第一次尝试](https://upload-images.jianshu.io/upload_images/3468445-77dce739598e20b1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 第二次尝试（对应hasStablelds情况，一般不用）
#### 还是从mAttachedScrap和mCacheView获取
```
final int type = mAdapter.getItemViewType(offsetPosition);
// 2) 如果StableIds存在的话，通过Stable Ids来获取
if (mAdapter.hasStableIds()) {
    holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
            type, dryRun);
    if (holder != null) {
        // 更新holder的位置
        holder.mPosition = offsetPosition;
        fromScrapOrHiddenOrCache = true;
    }
}
```
这里看到了`mAdapter.getItemViewType(offsetPosition)`方法，是在使用`RecyclerView`进行多类型`item`的方法。在前面第一次尝试中`mAttachedScrap`和`mCacheView`是没有区分**type**的，现在开始要区分**type**。

调用`adapter`的`setHasStableIds(true)`后，`mAdapter.hasStableIds()`返回`true`。
通过`getScrapOrCachedViewForId`方法获取
```
ViewHolder getScrapOrCachedViewForId(long id, int type, boolean dryRun) {
    // 首先在mAttachedScrap中寻找
    final int count = mAttachedScrap.size();
    for (int i = count - 1; i >= 0; i--) {
        final ViewHolder holder = mAttachedScrap.get(i);
        if (holder.getItemId() == id && !holder.wasReturnedFromScrap()) {
            if (type == holder.getItemViewType()) {
                holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
                if (holder.isRemoved()) {
                            // this might be valid in two cases:
                            // > item is removed but we are in pre-layout pass
                            // >> do nothing. return as is. make sure we don't rebind
                            // > item is removed then added to another position and we are in
                            // post layout.
                            // >> remove removed and invalid flags, add update flag to rebind
                            // because item was invisible to us and we don't know what happened in
                            // between.
                    if (!mState.isPreLayout()) {
                        holder.setFlags(ViewHolder.FLAG_UPDATE, ViewHolder.FLAG_UPDATE
                                | ViewHolder.FLAG_INVALID | ViewHolder.FLAG_REMOVED);
                    }
                }
                return holder;
            } else if (!dryRun) {
                // if we are running animations, it is actually better to keep it in scrap
                        // but this would force layout manager to lay it out which would be bad.
                        // Recycle this scrap. Type mismatch.
                mAttachedScrap.remove(i);
                removeDetachedView(holder.itemView, false);
                quickRecycleScrapView(holder.itemView);
            }
        }
    }

    // 从mCachedViews中寻找
    final int cacheSize = mCachedViews.size();
    for (int i = cacheSize - 1; i >= 0; i--) {
        final ViewHolder holder = mCachedViews.get(i);
        if (holder.getItemId() == id) {
            if (type == holder.getItemViewType()) {
                if (!dryRun) {
                    mCachedViews.remove(i);
                }
                return holder;
            } else if (!dryRun) {
                recycleCachedViewAt(i);
                return null;
            }
        }
    }
    return null;
}
```
比第一次尝试多了**type**和**id**的判断，也就是`adapter.setHasStableIds(true)`的话需要重写`adapter.getItemId`方法，为每个**ViewHolder**设置一个单独的**id**
![7866586-a325e2cb54705dec.png](https://upload-images.jianshu.io/upload_images/3468445-f4eab11d8c1e7d13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 第三次尝试（从自定义缓存中获取，一般不用）
在前面的整体代码中可以看到自定义缓存**mViewCacheExtension**相关代码
```
//自定义缓存，通过mViewCacheExtension进行获取
if (holder == null && mViewCacheExtension != null) {
    // We are NOT sending the offsetPosition because LayoutManager does not
    // know it.
    final View view = mViewCacheExtension
            .getViewForPositionAndType(this, position, type);
    if (view != null) {
        holder = getChildViewHolder(view);
        if (holder == null) {
            throw new IllegalArgumentException("getViewForPositionAndType returned"
                    + " a view which does not have a ViewHolder"
                    + exceptionLabel());
        } else if (holder.shouldIgnore()) {
            throw new IllegalArgumentException("getViewForPositionAndType returned"
                   + " a view that is ignored. You must call stopIgnoring before"
                   + " returning this view." + exceptionLabel());
        }
    }
}
```
要使用**mViewCacheExtension**需要使用`Recycler.setViewCacheExtension`方法，传入一个自定义**ViewCacheExtension**子类，对于这里暂时不去具体了解。

#### 第四次尝试（对应pool）
最后一级缓存是**Recycler**中的**mRecyclerPool**，是**RecycledViewPool**类型。这种缓存形式可以支持多个**RecyclerView**之间复用**View**。具体暂不分析。
#### 缓存中没有，自行创建

----------------------------------------------

## 回收
上面是复用缓存，现在是回收

&#160; &#160; &#160; &#160;在 `RecyclerView` 滑动时，会交由 `LinearLayoutManager` 的 `scrollVerticallyBy()` 去处理，然后 `LayoutManager` 会接着调用 `fill()` 方法去处理需要复用和回收的`item`，最终会调用` Recycler.recyclerView()` 这个方法开始进行回收工作。

&#160; &#160; &#160; &#160;`recyclerView()`之前还会调用`LayoutManager#removeViewAt`方法，-->ChildHelper#removeViewAt-->callback#removeViewAt（在RecyclerView中传入了callback实现类）-->RecyclerView.this.removeViewAt(index)-->ViewGroup#removeViewAt-->ViewGroup#removeViewInternal-->view.dispatchDetachedFromWindow()-->onDetachedFromWindow()
&#160; &#160; &#160; &#160;所以回收ViewHolder，也是会将View从RecyclerView这个ViewGroup中remove掉，跟平常的ViewGroup一样，同时也会是View调用`onDetachedFromWindow()`方法。**所以视图对象虽然会被回收复用，但它还是会先从容器中remove，也会调用onDetachedFromWindow方法。**