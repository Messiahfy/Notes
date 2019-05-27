通过`ItemDecoration`来实现类似联系人列表的`粘性头部`效果，如下图：
![效果图](https://upload-images.jianshu.io/upload_images/3468445-f39bc368acc71c5b.gif?imageMogr2/auto-orient/strip)

### 定义实体类
```
public class GroupItem {
    private String content;//item的内容字符串
    private boolean isFirst;//是否是该组第一项
    private String groupId;//所属组的id

    public GroupItem(String content, boolean isFirst, String groupId) {
        this.content = content;
        this.isFirst = isFirst;
        this.groupId = groupId;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public boolean isFirst() {
        return isFirst;
    }

    public void setFirst(boolean first) {
        isFirst = first;
    }

    public String getGroupId() {
        return groupId;
    }

    public void setGroupId(String groupId) {
        this.groupId = groupId;
    }
}
```
Deconation的代码如下：
```
public class HfyDecoration extends RecyclerView.ItemDecoration {
    private Paint paint;
    private GroupInterface groupInterface;

    public HfyDecoration(GroupInterface groupInterface) {
        paint = new Paint();
        paint.setColor(Color.RED);
        paint.setStyle(Paint.Style.FILL);
        paint.setAntiAlias(true);
        this.groupInterface = groupInterface;
    }

    @Override
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
        int index = parent.getChildAdapterPosition(view);
        if (groupInterface.isFirst(index)) {
            //如果是分组第一个，绘制高度为100px的标题
            outRect.set(0, 100, 0, 0);
        } else {
            //普通item，和分组首个的顶部撑开的空间不一样，只绘制5px的分割线
            outRect.set(0, 5, 0, 0);
        }
    }

    @Override
    public void onDraw(Canvas c, RecyclerView parent, RecyclerView.State state) {
        super.onDraw(c, parent, state);
        int childCount = parent.getChildCount();
        for (int i = 0; i < childCount; i++) {
            View view = parent.getChildAt(i);
            int index = parent.getChildAdapterPosition(view);
            if (groupInterface.isFirst(index)) {
                //如果是分组第一个，绘制标题
                float left = view.getLeft();
                float top = view.getTop() - 100;
                float bottom = view.getTop();
                float right = view.getRight();
                paint.setColor(Color.BLUE);
                c.drawRect(left, top, right, bottom, paint);
                paint.setColor(Color.RED);
                paint.setTextSize(100);
                c.drawText(groupInterface.getGroupId(index), left, bottom, paint);
            } else {
                //不是本组第一个，就绘制普通分割线
                float left = view.getLeft();
                float top = view.getTop() - 5;
                float bottom = view.getTop();
                float right = view.getRight();
                //绘制矩形作为分割线
                c.drawRect(left, top, right, bottom, paint);
            }

        }
    }

    @Override
    public void onDrawOver(Canvas c, RecyclerView parent, RecyclerView.State state) {
        super.onDrawOver(c, parent, state);
        //绘制顶部标题的坐标
        float left = parent.getLeft();
        float top = parent.getTop();
        float bottom = parent.getTop() + 100;
        float right = parent.getRight();

        int count = parent.getChildCount();

        //当前可见的第一个item
        View firstVisibleView = parent.getChildAt(0);
        //当前可见的第一个item在数据源中的index
        int firstVisbleIndex = parent.getChildAdapterPosition(firstVisibleView);
        //当前可见的第一个item的分组id
        String firstVisibleGroupId = groupInterface.getGroupId(firstVisbleIndex);
        //当前可见的第二个分组的第一个item在RecyclerView的可见位置，默认-1表示可见项没有第二个组
        int secondGroupFirstPosition = -1;
        //遍历当前可见的第一个之后的所有item（随着滑动，会不断调用，不断遍历实时的新的可见item）
        for (int i = 1; i < count; i++) {
            View item = parent.getChildAt(i);
            int index = parent.getChildAdapterPosition(item);
            if (!firstVisibleGroupId.equals(groupInterface.getGroupId(index))) {
                //如果和第一个可见item不是同一组，即当前可见的第二组的第一个item
                secondGroupFirstPosition = i;
                //找到了可见的第二组的第一个item，就停止循环
                break;
            }
        }
        //核心思想就是找到可见的第二个组的第一个view，如果这个view距顶部滑动到了两个标题的高度以内，
        // 即两个组的标题发生接触，就根据这个view的位置，修改绘制第一组的标题位置

        //如果secondGroupFirstPosition是-1，会返回null
        View secondGroupFirstView = parent.getChildAt(secondGroupFirstPosition);
        if (secondGroupFirstView != null && secondGroupFirstView.getTop() <= 200) {
            //第二组的第一个滑动到了特定距离,距顶部两个标题的高度，就开始跟随移动
            paint.setColor(Color.BLUE);
            //根据第二组的第一个view的top来设置第一组的标题位置，就产生了跟随滑动被顶上去的效果
            c.drawRect(left, secondGroupFirstView.getTop() - 200, right, secondGroupFirstView.getTop() - 100, paint);
            paint.setColor(Color.RED);
            //绘制文字同理
            c.drawText(groupInterface.getGroupId(firstVisbleIndex), left, secondGroupFirstView.getTop() - 100, paint);
        } else {
            //第二组的第一项没有滑到特定距离，或者可见只有一个组，就固定顶部显示第一组的标题id
            paint.setColor(Color.BLUE);
            c.drawRect(left, top, right, bottom, paint);
            paint.setColor(Color.RED);
            c.drawText(groupInterface.getGroupId(firstVisbleIndex), left, bottom, paint);
        }
    }

    public interface GroupInterface {

        boolean isFirst(int index);

        String getGroupId(int index);
    }
}
```
activity的代码如下：
```
public class MainActivity extends AppCompatActivity {

    private RecyclerView recyclerView;

    @Override
    protected void onCreate(final Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        recyclerView = findViewById(R.id.recycler_view);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));

        final ArrayList<GroupItem> list = new ArrayList<>();
        initData(list);
        //实现接口
        recyclerView.addItemDecoration(new HfyDecoration(new HfyDecoration.GroupInterface() {
            @Override
            public boolean isFirst(int index) {
                return list.get(index).isFirst();
            }

            @Override
            public String getGroupId(int index) {
                return list.get(index).getGroupId();
            }
        }));
        recyclerView.setAdapter(new HfyAdapter(list));
    }

    private void initData(ArrayList<GroupItem> list) {
        list.add(new GroupItem("aa", true, "a"));
        list.add(new GroupItem("aaa", false, "a"));
        list.add(new GroupItem("aaaa", false, "a"));
        list.add(new GroupItem("aaa", false, "a"));

        list.add(new GroupItem("bb", true, "b"));
        list.add(new GroupItem("bbb", false, "b"));
        list.add(new GroupItem("bbbb", false, "b"));
        list.add(new GroupItem("bbb", false, "b"));

        list.add(new GroupItem("cc", true, "c"));
        list.add(new GroupItem("ccc", false, "c"));
        list.add(new GroupItem("cccc", false, "c"));
        list.add(new GroupItem("ccc", false, "c"));

        list.add(new GroupItem("dd", true, "d"));
        list.add(new GroupItem("ddd", false, "d"));
        list.add(new GroupItem("dddd", false, "d"));
        list.add(new GroupItem("ddd", false, "d"));

        list.add(new GroupItem("ee", true, "e"));
        list.add(new GroupItem("eee", false, "e"));
        list.add(new GroupItem("eeee", false, "e"));
        list.add(new GroupItem("eee", false, "e"));

        list.add(new GroupItem("ff", true, "f"));
        list.add(new GroupItem("fff", false, "f"));
        list.add(new GroupItem("ffff", false, "f"));
        list.add(new GroupItem("fff", false, "f"));
        list.add(new GroupItem("fff", false, "f"));
        list.add(new GroupItem("fff", false, "f"));
        list.add(new GroupItem("fff", false, "f"));
        list.add(new GroupItem("fff", false, "f"));
        list.add(new GroupItem("fff", false, "f"));
        list.add(new GroupItem("fff", false, "f"));
        list.add(new GroupItem("fff", false, "f"));
        list.add(new GroupItem("fff", false, "f"));
        list.add(new GroupItem("fff", false, "f"));
        list.add(new GroupItem("fff", false, "f"));
        list.add(new GroupItem("fff", false, "f"));
    }
}
```