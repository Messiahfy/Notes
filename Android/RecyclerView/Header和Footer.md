## 1.实现方法
&#160; &#160; &#160; &#160;RecyclerView使用Adapter的getItemViewType(int position)方法，根据position返回不同的viewType，然后在Adapter的onCreateViewHolder中根据viewType创建不同的view。

熟悉了下拉刷新和上拉加载，再一并完成
还有多布局