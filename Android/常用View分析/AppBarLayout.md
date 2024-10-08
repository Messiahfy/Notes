AppBarLayout实现header嵌套滚动效果。layout_scrollFlags使用`scroll`。如果想要滚动只有展开和收缩两种状态，不停留在中间状态，需要使用`snap`；如果不想完全收缩，而是部分折叠，可以使用`exitUntilCollapsed`配合子view的minHeight
```
<?xml version="1.0" encoding="utf-8"?>
<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <com.google.android.material.appbar.AppBarLayout
        android:id="@+id/app_bar_layout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">

        <View
            android:layout_width="match_parent"
            android:layout_height="200dp"
            android:minHeight="100dp"
            app:layout_scrollFlags="scroll|exitUntilCollapsed|snap" />
    </com.google.android.material.appbar.AppBarLayout>

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recycler_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="com.google.android.material.appbar.AppBarLayout$ScrollingViewBehavior" />

</androidx.coordinatorlayout.widget.CoordinatorLayout>
```