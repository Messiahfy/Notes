https://github.com/hgDendi/ExpandableRecyclerView

```
package com.example.hfy.test.recycler;

import android.support.annotation.NonNull;
import android.support.v7.widget.RecyclerView;
import android.util.Log;
import android.view.View;
import android.view.ViewGroup;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.Set;


public abstract class BaseExpandableRecyclerViewAdapter
        <GroupBean extends BaseExpandableRecyclerViewAdapter.BaseGroupBean<ChildBean>,
                ChildBean,
                GroupViewHolder extends BaseExpandableRecyclerViewAdapter.BaseGroupViewHolder,
                ChildViewHolder extends RecyclerView.ViewHolder>
        extends RecyclerView.Adapter<RecyclerView.ViewHolder> {

    private static final String TAG = "BaseExpandableRecyclerV";

    private static final Object EXPAND_PAYLOAD = new Object();

    private static final int TYPE_EMPTY = ViewProducer.VIEW_TYPE_EMPTY;
    private static final int TYPE_HEADER = ViewProducer.VIEW_TYPE_HEADER;
    private static final int TYPE_GROUP = ViewProducer.VIEW_TYPE_EMPTY >> 2;
    private static final int TYPE_CHILD = ViewProducer.VIEW_TYPE_EMPTY >> 3;
    private static final int TYPE_MASK = TYPE_GROUP | TYPE_CHILD | TYPE_EMPTY | TYPE_HEADER;

    private Set<GroupBean> mExpandGroupSet;
    private Map<GroupBean, BaseGroupViewHolder> mGroupBeanVideoHolder;
    private ExpandableRecyclerViewOnClickListener<GroupBean, ChildBean> mListener;

    private boolean mIsEmpty;
    private boolean mShowHeaderViewWhenEmpty;
    private ViewProducer mEmptyViewProducer;
    private ViewProducer mHeaderViewProducer;

    public GroupViewHolder getGroupViewHolder(GroupBean bean) {
        return (GroupViewHolder) mGroupBeanVideoHolder.get(bean);
    }

    public boolean isExpand(GroupBean groupBean) {
        return mExpandGroupSet.contains(groupBean);
    }

    public BaseExpandableRecyclerViewAdapter() {
        mExpandGroupSet = new HashSet<>();
        registerAdapterDataObserver(new RecyclerView.AdapterDataObserver() {
            @Override
            public void onChanged() {
                // after notifyDataSetChange(),clear outdated list
                List<GroupBean> retainItem = new ArrayList<>();
                for (int i = 0; i < getGroupCount(); i++) {
                    GroupBean groupBean = getGroupItem(i);
                    if (mExpandGroupSet.contains(groupBean)) {
                        retainItem.add(groupBean);
                    }
                }
                mExpandGroupSet.clear();
                mExpandGroupSet.addAll(retainItem);
            }
        });

        mGroupBeanVideoHolder = new HashMap<>();
    }

    /**
     * get group count
     *
     * @return group count
     */
    abstract public int getGroupCount();

    /**
     * get groupItem related to GroupCount
     *
     * @param groupIndex the index of group item in group list
     * @return related GroupBean
     */
    abstract public GroupBean getGroupItem(int groupIndex);

    protected int getGroupType(GroupBean groupBean) {
        return 0;
    }

    /**
     * create {@link GroupViewHolder} for group item
     *
     * @param parent
     * @return
     */
    abstract public GroupViewHolder onCreateGroupViewHolder(ViewGroup parent, int groupViewType);

    /**
     * bind {@link GroupViewHolder}
     *
     * @param holder
     * @param groupBean
     * @param isExpand
     */
    abstract public void onBindGroupViewHolder(GroupViewHolder holder, GroupBean groupBean, boolean isExpand);

    /**
     * bind {@link GroupViewHolder} with payload , used to invalidate partially
     *
     * @param holder
     * @param groupBean
     * @param isExpand
     * @param payload
     */
    protected void onBindGroupViewHolder(GroupViewHolder holder, GroupBean groupBean, boolean isExpand, List<Object> payload) {
        onBindGroupViewHolder(holder, groupBean, isExpand);
    }

    protected int getChildType(GroupBean groupBean, ChildBean childBean) {
        return 0;
    }

    /**
     * create {@link ChildViewHolder} for child item
     *
     * @param parent
     * @return
     */
    abstract public ChildViewHolder onCreateChildViewHolder(ViewGroup parent, int childViewType);

    /**
     * bind {@link ChildViewHolder}
     *
     * @param holder
     * @param groupBean
     * @param childBean
     */
    abstract public void onBindChildViewHolder(ChildViewHolder holder, GroupBean groupBean, ChildBean childBean);


    /**
     * bind {@link ChildViewHolder} with payload , used to invalidate partially
     *
     * @param holder
     * @param groupBean
     * @param childBean
     * @param payload
     */
    protected void onBindChildViewHolder(ChildViewHolder holder, GroupBean groupBean, ChildBean childBean, List<Object> payload) {
        onBindChildViewHolder(holder, groupBean, childBean);
    }


    public void setEmptyViewProducer(ViewProducer emptyViewProducer) {
        if (mEmptyViewProducer != emptyViewProducer) {
            mEmptyViewProducer = emptyViewProducer;
            if (mIsEmpty) {
                notifyDataSetChanged();
            }
        }
    }

    public void setHeaderViewProducer(ViewProducer headerViewProducer, boolean showWhenEmpty) {
        mShowHeaderViewWhenEmpty = showWhenEmpty;
        if (mHeaderViewProducer != headerViewProducer) {
            mHeaderViewProducer = headerViewProducer;
            notifyDataSetChanged();
        }
    }

    public final void setListener(ExpandableRecyclerViewOnClickListener<GroupBean, ChildBean> listener) {
        mListener = listener;
    }

    public final boolean isGroupExpanding(GroupBean groupBean) {
        return mExpandGroupSet.contains(groupBean);
    }

    public final boolean expandGroup(GroupBean groupBean) {
        if (groupBean.isExpandable() && !isGroupExpanding(groupBean)) {
            mExpandGroupSet.add(groupBean);
            final int position = getAdapterPosition(getGroupIndex(groupBean));
            notifyItemRangeInserted(position + 1, groupBean.getChildCount());
            notifyItemChanged(position, EXPAND_PAYLOAD);
            return true;
        }
        return false;
    }

    public final void foldAll() {
        Iterator<GroupBean> iter = mExpandGroupSet.iterator();
        while (iter.hasNext()) {
            GroupBean groupBean = iter.next();
            final int position = getAdapterPosition(getGroupIndex(groupBean));
            notifyItemRangeRemoved(position + 1, groupBean.getChildCount());
            notifyItemChanged(position, EXPAND_PAYLOAD);
            iter.remove();
        }
    }

    public final boolean foldGroup(GroupBean groupBean) {
        if (mExpandGroupSet.remove(groupBean)) {
            final int position = getAdapterPosition(getGroupIndex(groupBean));
            notifyItemRangeRemoved(position + 1, groupBean.getChildCount());
            notifyItemChanged(position, EXPAND_PAYLOAD);
            return true;
        }
        return false;
    }

    @Override
    public final int getItemCount() {
        int result = getGroupCount();
        if (result == 0 && mEmptyViewProducer != null) {
            mIsEmpty = true;
            return mHeaderViewProducer != null && mShowHeaderViewWhenEmpty ? 2 : 1;
        }
        mIsEmpty = false;
        for (GroupBean groupBean : mExpandGroupSet) {
            if (getGroupIndex(groupBean) < 0) {
                Log.e(TAG, "invalid index in expandgroupList : " + groupBean);
                continue;
            }
            result += groupBean.getChildCount();
        }
        if (mHeaderViewProducer != null) {
            result++;
        }
        return result;
    }

    public final int getAdapterPosition(int groupIndex) {
        int result = groupIndex;
        for (GroupBean groupBean : mExpandGroupSet) {
            if (getGroupIndex(groupBean) >= 0 && getGroupIndex(groupBean) < groupIndex) {
                result += groupBean.getChildCount();
            }
        }
        if (mHeaderViewProducer != null) {
            result++;
        }
        return result;
    }

    public final int getGroupIndex(@NonNull GroupBean groupBean) {
        for (int i = 0; i < getGroupCount(); i++) {
            if (groupBean.equals(getGroupItem(i))) {
                return i;
            }
        }
        return -1;
    }

    @Override
    public final int getItemViewType(int position) {
        if (mIsEmpty) {
            return position == 0 && mShowHeaderViewWhenEmpty && mHeaderViewProducer != null ? TYPE_HEADER : TYPE_EMPTY;
        }
        if (position == 0 && mHeaderViewProducer != null) {
            return TYPE_HEADER;
        }
        int[] coord = translateToDoubleIndex(position);
        GroupBean groupBean = getGroupItem(coord[0]);
        if (coord[1] < 0) {
            int groupType = getGroupType(groupBean);
            if ((groupType & TYPE_MASK) == 0) {
                return groupType | TYPE_GROUP;
            } else {
                throw new IllegalStateException(
                        String.format(Locale.getDefault(), "GroupType [%d] conflits with MASK [%d]", groupType, TYPE_MASK));
            }
        } else {
            int childType = getChildType(groupBean, groupBean.getChildAt(coord[1]));
            if ((childType & TYPE_MASK) == 0) {
                return childType | TYPE_CHILD;
            } else {
                throw new IllegalStateException(
                        String.format(Locale.getDefault(), "ChildType [%d] conflits with MASK [%d]", childType, TYPE_MASK));
            }
        }
    }


    @Override
    public final RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        switch (viewType & TYPE_MASK) {
            case TYPE_EMPTY:
                return mEmptyViewProducer.onCreateViewHolder(parent);
            case TYPE_HEADER:
                return mHeaderViewProducer.onCreateViewHolder(parent);
            case TYPE_CHILD:
                return onCreateChildViewHolder(parent, viewType ^ TYPE_CHILD);
            case TYPE_GROUP:
                return onCreateGroupViewHolder(parent, viewType ^ TYPE_GROUP);
            default:
                throw new IllegalStateException(
                        String.format(Locale.getDefault(), "Illegal view type : viewType[%d]", viewType));

        }
    }

    @Override
    public final void onBindViewHolder(final RecyclerView.ViewHolder holder, int position) {
        onBindViewHolder(holder, position, null);
    }

    @Override
    public final void onBindViewHolder(RecyclerView.ViewHolder holder, int position, List<Object> payloads) {
        switch (holder.getItemViewType() & TYPE_MASK) {
            case TYPE_EMPTY:
                mEmptyViewProducer.onBindViewHolder(holder);
                break;
            case TYPE_HEADER:
                mHeaderViewProducer.onBindViewHolder(holder);
                break;
            case TYPE_CHILD:
                final int[] childCoord = translateToDoubleIndex(position);
                GroupBean groupBean = getGroupItem(childCoord[0]);
                bindChildViewHolder((ChildViewHolder) holder, groupBean, groupBean.getChildAt(childCoord[1]), payloads);
                break;
            case TYPE_GROUP:
                bindGroupViewHolder((GroupViewHolder) holder, getGroupItem(translateToDoubleIndex(position)[0]), payloads);
                mGroupBeanVideoHolder.put(getGroupItem(translateToDoubleIndex(position)[0]), (GroupViewHolder) holder);
                break;
            default:
                throw new IllegalStateException(
                        String.format(Locale.getDefault(), "Illegal view type : position [%d] ,itemViewType[%d]", position, holder.getItemViewType()));
        }
    }

    protected void bindGroupViewHolder(final GroupViewHolder holder, final GroupBean groupBean, List<Object> payload) {
        if (payload != null && payload.size() != 0) {
            if (payload.contains(EXPAND_PAYLOAD)) {
                holder.onExpandStatusChanged(BaseExpandableRecyclerViewAdapter.this, isGroupExpanding(groupBean));
                if (payload.size() == 1) {
                    return;
                }
            }
            onBindGroupViewHolder(holder, groupBean, isGroupExpanding(groupBean), payload);
            return;
        }
        holder.itemView.setOnLongClickListener(new View.OnLongClickListener() {
            @Override
            public boolean onLongClick(View v) {
                if (mListener != null) {
                    return mListener.onGroupLongClicked(groupBean);
                }
                return false;
            }
        });
        if (!groupBean.isExpandable()) {
            holder.itemView.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    if (mListener != null) {
                        mListener.onGroupClicked(groupBean);
                    }
                }
            });
        } else {
            holder.itemView.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    mListener.onGroupClicked(groupBean);
                    final boolean isExpand = mExpandGroupSet.contains(groupBean);
                    //点击了同一个
                    if (isExpand) {
                        if (mListener == null || !mListener.onInterceptGroupExpandEvent(groupBean, isExpand)) {
                            final int adapterPosition = holder.getAdapterPosition();
                            holder.onExpandStatusChanged(BaseExpandableRecyclerViewAdapter.this, !isExpand);
                            mExpandGroupSet.remove(groupBean);
                            notifyItemRangeRemoved(adapterPosition + 1, groupBean.getChildCount());
                        }
                    } else {
                        // 关掉之前打开了的
                        for (GroupBean bean : mExpandGroupSet) {
                            BaseGroupViewHolder holder1 = mGroupBeanVideoHolder.get(bean);
                            if (holder1 != null) {
                                int adapterPosition = holder1.getAdapterPosition();
                                notifyItemRangeRemoved(adapterPosition + 1, bean.getChildCount());
                            }
                        }
                        mExpandGroupSet.clear();
                        mExpandGroupSet.add(groupBean);
                        int adapterPosition = holder.getAdapterPosition();
                        notifyItemRangeInserted(adapterPosition + 1, groupBean.getChildCount());
                    }
                }
            });
        }
        onBindGroupViewHolder(holder, groupBean, isGroupExpanding(groupBean));
    }

    protected void bindChildViewHolder(ChildViewHolder holder, final GroupBean groupBean, final ChildBean childBean, List<Object> payload) {
        onBindChildViewHolder(holder, groupBean, childBean, payload);
        holder.itemView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (mListener != null) {
                    mListener.onChildClicked(groupBean, childBean);
                }
            }
        });
    }

    /**
     * position translation
     * from adapterPosition to group-child coord
     *
     * @param adapterPosition adapterPosition
     * @return int[]{groupIndex,childIndex}
     */
    protected final int[] translateToDoubleIndex(int adapterPosition) {
        if (mHeaderViewProducer != null) {
            adapterPosition--;
        }
        final int[] result = new int[]{-1, -1};
        final int groupCount = getGroupCount();
        int adaptePositionCursor = 0;
        for (int groupCursor = 0; groupCursor < groupCount; groupCursor++) {
            if (adaptePositionCursor == adapterPosition) {
                result[0] = groupCursor;
                result[1] = -1;
                break;
            }
            GroupBean groupBean = getGroupItem(groupCursor);
            if (mExpandGroupSet.contains(groupBean)) {
                final int childCount = groupBean.getChildCount();
                final int offset = adapterPosition - adaptePositionCursor;
                if (childCount >= offset) {
                    result[0] = groupCursor;
                    result[1] = offset - 1;
                    break;
                }
                adaptePositionCursor += childCount;
            }
            adaptePositionCursor++;
        }
        return result;
    }


    public interface BaseGroupBean<ChildBean> {
        /**
         * get num of children
         *
         * @return
         */
        int getChildCount();

        /**
         * get child at childIndex
         *
         * @param childIndex integer between [0,{@link #getChildCount()})
         * @return
         */
        ChildBean getChildAt(int childIndex);

        /**
         * whether this BaseGroupBean is expandable
         *
         * @return
         */
        boolean isExpandable();
    }

    public static abstract class BaseGroupViewHolder extends RecyclerView.ViewHolder {
        public BaseGroupViewHolder(View itemView) {
            super(itemView);
        }

        /**
         * optimize for partial invalidate,
         * when switching fold status.
         * Default implementation is update the whole {android.support.v7.widget.RecyclerView.ViewHolder#itemView}.
         * <p>
         * Warning:If the itemView is invisible , the callback will not be called.
         *
         * @param relatedAdapter
         * @param isExpanding
         */
        protected abstract void onExpandStatusChanged(RecyclerView.Adapter relatedAdapter, boolean isExpanding);
    }


    public interface ExpandableRecyclerViewOnClickListener<GroupBean extends BaseGroupBean, ChildBean> {

        /**
         * called when group item is long clicked
         *
         * @param groupItem
         * @return
         */
        boolean onGroupLongClicked(GroupBean groupItem);

        /**
         * called when an expandable group item is clicked
         *
         * @param groupItem
         * @param isExpand
         * @return whether intercept the click event
         */
        boolean onInterceptGroupExpandEvent(GroupBean groupItem, boolean isExpand);

        /**
         * called when an unexpandable group item is clicked
         *
         * @param groupItem
         */
        void onGroupClicked(GroupBean groupItem);

        /**
         * called when child is clicked
         *
         * @param groupItem
         * @param childItem
         */
        void onChildClicked(GroupBean groupItem, ChildBean childItem);
    }
}
```

```
package com.example.hfy.test.recycler;

import android.support.v7.widget.RecyclerView;
import android.view.View;
import android.view.ViewGroup;

public interface ViewProducer {
    int VIEW_TYPE_EMPTY = 1 << 30;
    int VIEW_TYPE_HEADER = VIEW_TYPE_EMPTY >> 1;

    /**
     * equivalent to RecyclerView.Adapter#onCreateViewHolder(ViewGroup, int)
     *
     * @param parent
     * @return
     */
    RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent);

    /**
     * equivalent to RecyclerView.Adapter#onBindViewHolder(RecyclerView.ViewHolder, int)
     *
     * @param holder
     */
    void onBindViewHolder(RecyclerView.ViewHolder holder);

    public static class DefaultEmptyViewHolder extends RecyclerView.ViewHolder {
        public DefaultEmptyViewHolder(View itemView) {
            super(itemView);
        }
    }
}
```

```
package com.example.hfy.test;

import android.os.Bundle;
import android.support.v7.app.AppCompatActivity;
import android.support.v7.widget.LinearLayoutManager;
import android.support.v7.widget.RecyclerView;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.TextView;

import com.example.hfy.test.recycler.BaseExpandableRecyclerViewAdapter;

import java.util.ArrayList;
import java.util.List;

public class MainActivity extends AppCompatActivity {
    private RecyclerView recyclerView;
    private MainExpandableAdapter mAdapter;

    @Override
    protected void onCreate(final Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        recyclerView = findViewById(R.id.recycler_view);
        recyclerView.setItemAnimator(null);
        List<GroupBean> groupBeans = initGroupData();
        recyclerView.setLayoutManager(new LinearLayoutManager(this));
        mAdapter = new MainExpandableAdapter(groupBeans);
        mAdapter.setListener(new BaseExpandableRecyclerViewAdapter.ExpandableRecyclerViewOnClickListener<GroupBean, ChildBean>() {
            @Override
            public boolean onGroupLongClicked(GroupBean groupItem) {
                return false;
            }

            @Override
            public boolean onInterceptGroupExpandEvent(GroupBean groupItem, boolean isExpand) {
                return false;
            }

            @Override
            public void onGroupClicked(GroupBean groupItem) {

            }

            @Override
            public void onChildClicked(GroupBean groupItem, ChildBean childItem) {

            }
        });
        recyclerView.setAdapter(mAdapter);
    }

    private List<GroupBean> initGroupData() {
        List<GroupBean> groupList = new ArrayList<>();


        // 直播
        List<ChildBean> pusherChildList = new ArrayList<>();
        pusherChildList.add(new ChildBean("美女直播"));
        if (pusherChildList.size() != 0) {
            // 这个是网页链接，配合build.sh避免在如ugc_smart版中出现
            pusherChildList.add(new ChildBean("小直播"));
            GroupBean pusherGroupBean = new GroupBean("直播", pusherChildList);
            groupList.add(pusherGroupBean);
        }

        // 初始化播放器
        List<ChildBean> playerChildList = new ArrayList<>();
        playerChildList.add(new ChildBean("超级播放器"));
        if (playerChildList.size() != 0) {
            GroupBean playerGroupBean = new GroupBean("播放器", playerChildList);
            groupList.add(playerGroupBean);
        }

        // 短视频
        List<ChildBean> shortVideoChildList = new ArrayList<>();

        if (shortVideoChildList.size() != 0) {
            // 这个是网页链接，配合build.sh避免在其他版本中出现
            shortVideoChildList.add(new ChildBean("小视频"));
            GroupBean shortVideoGroupBean = new GroupBean("短视频", shortVideoChildList);
            groupList.add(shortVideoGroupBean);
        }

        // 视频通话
        List<ChildBean> videoConnectChildList = new ArrayList<>();
        videoConnectChildList.add(new ChildBean("双人音视频"));
        videoConnectChildList.add(new ChildBean("多人音视频"));
        videoConnectChildList.add(new ChildBean("多人音视频"));
        videoConnectChildList.add(new ChildBean("多人音视频"));
        videoConnectChildList.add(new ChildBean("多人音视频"));
        videoConnectChildList.add(new ChildBean("多人音视频"));
        videoConnectChildList.add(new ChildBean("多人音视频"));
        videoConnectChildList.add(new ChildBean("多人音视频"));
        videoConnectChildList.add(new ChildBean("多人音视频"));
        videoConnectChildList.add(new ChildBean("多人音视频"));
        videoConnectChildList.add(new ChildBean("多人音视频"));
        if (videoConnectChildList.size() != 0) {
            GroupBean videoConnectGroupBean = new GroupBean("视频通话", videoConnectChildList);
            groupList.add(videoConnectGroupBean);
        }


        // 调试工具
        List<ChildBean> debugChildList = new ArrayList<>();
        debugChildList.add(new ChildBean("RTMP 推流"));
        debugChildList.add(new ChildBean("直播播放器"));
        debugChildList.add(new ChildBean("点播播放器"));
        debugChildList.add(new ChildBean("WebRTC Room"));
        if (debugChildList.size() != 0) {
            GroupBean debugGroupBean = new GroupBean("调试工具", debugChildList);
            groupList.add(debugGroupBean);
        }

        return groupList;
    }

    private static class MainExpandableAdapter extends BaseExpandableRecyclerViewAdapter<GroupBean, ChildBean, MainExpandableAdapter.GroupVH, MainExpandableAdapter.ChildVH> {
        private List<GroupBean> mListGroupBean;


        public MainExpandableAdapter(List<GroupBean> list) {
            mListGroupBean = list;
        }

        @Override
        public int getGroupCount() {
            return mListGroupBean.size();
        }

        @Override
        public GroupBean getGroupItem(int groupIndex) {
            return mListGroupBean.get(groupIndex);
        }

        @Override
        public GroupVH onCreateGroupViewHolder(ViewGroup parent, int groupViewType) {
            return new GroupVH(LayoutInflater.from(parent.getContext()).inflate(R.layout.item_group, parent, false));
        }

        @Override
        public void onBindGroupViewHolder(GroupVH holder, GroupBean groupBean, boolean isExpand) {
            holder.textView.setText(groupBean.mName);
        }

        @Override
        public ChildVH onCreateChildViewHolder(ViewGroup parent, int childViewType) {
            return new ChildVH(LayoutInflater.from(parent.getContext()).inflate(R.layout.item_child, parent, false));
        }

        @Override
        public void onBindChildViewHolder(ChildVH holder, GroupBean groupBean, ChildBean childBean) {
            holder.textView.setText(childBean.getName());
        }

        public class GroupVH extends BaseExpandableRecyclerViewAdapter.BaseGroupViewHolder {
            TextView textView;

            GroupVH(View itemView) {
                super(itemView);
                textView = itemView.findViewById(R.id.tv_name);
            }

            @Override
            protected void onExpandStatusChanged(RecyclerView.Adapter relatedAdapter, boolean isExpanding) {
            }

        }

        public class ChildVH extends RecyclerView.ViewHolder {
            TextView textView;

            ChildVH(View itemView) {
                super(itemView);
                textView = itemView.findViewById(R.id.tv_name);
            }
        }
    }

    private class GroupBean implements BaseExpandableRecyclerViewAdapter.BaseGroupBean<ChildBean> {
        private String mName;
        private List<ChildBean> mChildList;

        public GroupBean(String name, List<ChildBean> list) {
            mName = name;
            mChildList = list;
        }

        @Override
        public int getChildCount() {
            return mChildList.size();
        }

        @Override
        public ChildBean getChildAt(int index) {
            return mChildList.size() <= index ? null : mChildList.get(index);
        }

        @Override
        public boolean isExpandable() {
            return getChildCount() > 0;
        }

        public String getName() {
            return mName;
        }

        public List<ChildBean> getChildList() {
            return mChildList;
        }

    }

    private class ChildBean {
        public String mName;

        public ChildBean(String mName) {
            this.mName = mName;
        }

        public String getName() {
            return mName;
        }
    }
}
```

## 杭州自写代码关键部分
```
package com.lingshi.qingshuocustomerservice.module.organization.adapter;

import android.view.View;

import com.lingshi.qingshuocustomerservice.utils.EmptyUtils;
import com.lingshi.qingshuocustomerservice.widget.recycler.adapter.Entry;
import com.lingshi.qingshuocustomerservice.widget.recycler.adapter.FasterHolder;
import com.lingshi.qingshuocustomerservice.widget.recycler.adapter.Strategy;

import java.util.ArrayList;
import java.util.List;

/**
 * 用于RecyclerView分组列表
 *
 * @param <GroupBean>
 */
public abstract class BaseGroupStrategy<GroupBean extends BaseGroupStrategy.BaseGroupBean<?>> extends Strategy<GroupBean> {
    private static final Object EXPAND_PAYLOAD = new Object();

    @Override
    public final void onBindViewHolder(FasterHolder holder, GroupBean data, List<Object> payloads) {
        if (EmptyUtils.isEmpty(payloads)) {
            holder.itemView.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    /*
                    当adapter更新了数据源，调用notify...方法，会调用requestLayout()方法请求重新布局绘制视图，
                    但是要等到下一个Vsync信号到来才会开始执行[测量、布局（涉及到onBindViewHolder的执行）、绘制流程]，
                    那么在下一个Vsync信号到来前，数据源已经和之前不一样，但是新一轮onBindViewHolder方法还没有执行，
                    这样会造成这里点击监听器引用的data还是数据更新前的数据，而holder.getAdapter().getEntryList()已经变了。
                    所以要先判断data和数据源中的是否是同一个对象，如果不是就说明数据源改变而还没有刷新视图
                    避免在这段时间点击分组发生错误（也可以使用对adapter调用registerAdapterDataObserver()方法来监听数据源改变
                    ，从而作出应对）
                    */
                    if (data != holder.getAdapter().getListItem(holder.getListPosition())) {
                        return;
                    }
                    if (data.isExpanded()) {
                        //已展开，则关闭分组
                        data.setExpanded(false);
                        List<Entry<Object>> list = holder.getAdapter().getEntryList();
                        for (int i = holder.getListPosition() + 1; i < holder.getListPosition() + 1 + data.getChildCount(); i++) {
                            list.remove(holder.getListPosition() + 1);
                        }
                        holder.getAdapter().notifyItemRangeRemoved(holder.getAdapterPosition() + 1, data.getChildCount());
                    } else {
                        //展开分组
                        data.setExpanded(true);
                        List<Object> list = new ArrayList<>();
                        for (int i = 0; i < data.getChildCount(); i++) {
                            list.add(data.getChildAt(i));
                        }
                        holder.getAdapter().addAllSource(holder.getListPosition() + 1, list);
                    }
                    holder.getAdapter().notifyItemChanged(holder.getAdapterPosition(), EXPAND_PAYLOAD);
                }
            });
            onBindViewHolder(holder, data);
        } else {
            onBindPartialViewHolder(holder, data, payloads);
        }
    }

    /**
     * 局部刷新，用于分组展开和关闭的视图改变，例如箭头方向
     *
     * @param holder
     * @param data
     * @param payloads
     */
    protected abstract void onBindPartialViewHolder(FasterHolder holder, GroupBean data, List<Object> payloads);


    public abstract static class BaseGroupBean<ChildBean> {
        /**
         * 标记当前分组是否展开
         */
        private boolean isExpanded;

        /**
         * 获取组内 children 的数量
         *
         * @return
         */
        public abstract int getChildCount();

        /**
         * 获得 childIndex 位置的 child
         *
         * @param childIndex 在 [0,{@link #getChildCount()}) 之间的整数
         * @return 返回该位置的 child
         */
        public abstract ChildBean getChildAt(int childIndex);

        /**
         * 分组当前是否已经展开
         *
         * @return
         */
        public final boolean isExpanded() {
            return isExpanded;
        }

        /**
         * 设置是否已经展开的标记（如果要在初始显示就展开分组，就调用方法传true，但要一定自行把它的children添加到列表 ）
         *
         * @param expanded
         */
        public final void setExpanded(boolean expanded) {
            isExpanded = expanded;
        }
    }
}

```
```
package com.lingshi.qingshuocustomerservice.module.organization.adapter;

import android.text.TextUtils;
import android.view.View;
import android.widget.TextView;

import com.lingshi.qingshuocustomerservice.R;
import com.lingshi.qingshuocustomerservice.module.bean.OrganizationBean;
import com.lingshi.qingshuocustomerservice.widget.recycler.adapter.FasterHolder;
import com.lingshi.qingshuocustomerservice.widget.recycler.adapter.Strategy;

public class OrganizationChildStrategy extends Strategy<OrganizationBean.ArrayBean> {
    private OnMemberItemClickListener onMemberItemClickListener;
    private View.OnClickListener onClickListener;

    @Override
    public int layoutId() {
        return R.layout.include_chat_organization;
    }

    @Override
    public void onBindViewHolder(FasterHolder holder, OrganizationBean.ArrayBean data) {
        if (!TextUtils.isEmpty(data.getPhotoUrl())) {
            holder.setImage(R.id.iv_avatar, data.getPhotoUrl());
        }
        holder.setText(R.id.contact_name, data.getNickname());

        /*
        另一种优化方式：也可以使用RecyclerView.addOnItemTouchListener()，配合GestureDetector和
        RecyclerView.findChildViewUnder(e.getX(), e.getY())找到点击位置的View，然后通过
        RecyclerView.getChildViewHolder(view)得到点击的ViewHolder，然后自定义操作
        */

        // 优化RecyclerView，使用view.setTag绑定信息，避免每个item都创建一个 onClickListener
        if (onClickListener == null) {
            onClickListener = new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    if (onMemberItemClickListener != null) {
                        onMemberItemClickListener.onMemberItemClick((int) v.getTag(R.id.key_position), (OrganizationBean.ArrayBean) v.getTag(R.id.key_data));
                    }
                }
            };
        }
        holder.setOnClickListener(R.id.ll_container, onClickListener);
        holder.findViewById(R.id.ll_container).setTag(R.id.key_position, holder.getListPosition());
        holder.findViewById(R.id.ll_container).setTag(R.id.key_data, data);

        TextView onlineStatus = holder.findViewById(R.id.online_status);
        if (data.getOnline() == 2) {
            //在线
            onlineStatus.setText("在线");
            onlineStatus.setTextColor(holder.getContext().getResources().getColor(R.color.blue));
            onlineStatus.setSelected(true);
        } else {
            //离线
            onlineStatus.setText("离线");
            onlineStatus.setTextColor(holder.getContext().getResources().getColor(R.color.rose));
            onlineStatus.setSelected(false);
        }
    }

    public interface OnMemberItemClickListener {
        void onMemberItemClick(int position, OrganizationBean.ArrayBean data);
    }

    public void setOnMemberItemClickListener(OnMemberItemClickListener onMemberItemClickListener) {
        this.onMemberItemClickListener = onMemberItemClickListener;
    }
}

```