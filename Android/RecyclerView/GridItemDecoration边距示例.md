```
class GridItemDecoration(private val margin: Int) : RecyclerView.ItemDecoration() {

    override fun getItemOffsets(
        outRect: Rect,
        view: View,
        parent: RecyclerView,
        state: RecyclerView.State
    ) {
        super.getItemOffsets(outRect, view, parent, state)
        val gridLayoutManager: GridLayoutManager = parent.layoutManager as GridLayoutManager
        val childCount = gridLayoutManager.itemCount
        val spanSizeLookup = gridLayoutManager.spanSizeLookup
        val spanCount = gridLayoutManager.spanCount
        //注释掉的方式是用的viewPosition，改为下面的adapterPosition，避免动画时计算使用的还是旧位置
//        val spanIndex = (view.layoutParams as GridLayoutManager.LayoutParams).spanIndex
//        val spanSize = (view.layoutParams as GridLayoutManager.LayoutParams).spanSize
//        val position = gridLayoutManager.getPosition(view)
        val position = parent.getChildAdapterPosition(view)
        val spanIndex = gridLayoutManager.spanSizeLookup.getSpanIndex(position, spanCount)
        val spanSize = gridLayoutManager.spanSizeLookup.getSpanSize(position)

        var left = 0
        var top = 0
        var right = 0
        var bottom = 0

        // 设置水平间距
        when {
            spanIndex == 0 -> {
                //第一列
                right = margin / 2
            }
            spanIndex + spanSize == spanCount -> {
                //最后一列
                left = margin / 2
            }
            spanCount - spanIndex < spanSizeLookup.getSpanIndex(position + 1, spanCount) -> {
                //最后一列（没有占满一行的情况）
                left = margin / 2
            }
            else -> {
                // 中间列
                left = margin / 2
                right = margin / 2
            }
        }

        // 设置垂直间距
        val currentRowCount = getCurrentRowCount(position, spanSizeLookup, spanCount)
        val totalRowCount = getCurrentRowCount(childCount, spanSizeLookup, spanCount)
        when (currentRowCount) {
            0 -> {
                // 第一行
                bottom = margin / 2
            }
            totalRowCount -> {
                // 最后一行
                top = margin / 2
            }
            else -> {
                // 中间行
                top = margin / 2
                bottom = margin / 2
            }
        }
        outRect.set(left, top, right, bottom)
    }

    private fun getCurrentRowCount(
        position: Int,
        spanSizeLookup: GridLayoutManager.SpanSizeLookup,
        spanCount: Int
    ): Int {
        var span = 0
        var group = 0
        for (i in 0..position) {
            val size = spanSizeLookup.getSpanSize(i)
            span += size
            if (span > spanCount) {
                span = size
                group++
            }
        }
        return group
    }
}
```