# Fragment
&emsp;&emsp;Fragment 表示 Activity 中的行为或用户界面部分。您可以将多个片段组合在一个 Activity 中来构建多窗格 UI，以及在多个 Activity 中重复使用某个片段。您可以
将片段视为 Activity 的模块化组成部分，它具有自己的生命周期，能接收自己的输入事件，并且您可以在 Activity 运行时添加或移除片段（有点像您可以在不同 Activity 
中重复使用的“子 Activity”）。  

&emsp;&emsp;片段必须始终嵌入在 Activity 中，其生命周期直接受宿主 Activity 生命周期的影响。 例如，当 Activity 暂停时，其中的所有片段也会暂停；当
Activity 被销毁时，所有片段也会被销毁。 不过，当 Activity 正在运行（处于resumed生命周期状态）时，您可以独立操纵每个片段，如添加或移除它们。 当您执行此
类片段事务时，您也可以将其添加到由 Activity 管理的返回栈 — Activity 中的每个返回栈条目都是一条已发生片段事务的记录。 返回栈让用户可以通过按返回按钮撤消
片段事务（后退）。  

&emsp;&emsp;当您将片段作为 Activity 布局的一部分添加时，它存在于 Activity 视图层次结构的某个 ViewGroup 内部，并且片段会定义其自己的视图布局。您可以通过在 Activity
的布局文件中声明片段，将其作为 \<fragment\> 元素插入您的 Activity 布局中，或者通过将其添加到某个现有 ViewGroup，利用应用代码进行插入。不过，片段并非必须
成为 Activity 布局的一部分；您还可以将没有自己 UI 的片段用作 Activity 的不可见工作线程。

## 创建Fragment
&emsp;&emsp;要想创建片段，您必须创建 Fragment（或已有其子类）的子类。Fragment 类的代码与 Activity 非常相似。它包含与 Activity 类似的回调方法，如 onCreate()、
onStart()、onPause() 和 onStop()等。  

&emsp;&emsp;通常，应该实现某些生命周期回调方法。  
&emsp;&emsp;如果想扩展某些子类，而不是Fragment基类：有**DialogFragment**、**ListFragment**和**PreferenceFragment**。
* DialogFragment
&emsp;&emsp;显示浮动对话框。使用此类创建对话框可有效地替代使用 Activity 类中的对话框帮助程序方法，因为您可以将片段对话框纳入由 Activity 管理的片段返回
栈，从而使用户能够返回清除的片段。
* ListFragment
&emsp;&emsp;显示由适配器（如 SimpleCursorAdapter）管理的一系列项目，类似于 ListActivity。它提供了几种管理列表视图的方法，如用于处理点击事件的 onListItemClick() 回调。
* PreferenceFragment
&emsp;&emsp;以列表形式显示 Preference 对象的层次结构，类似于 PreferenceActivity。这在为您的应用创建“设置” Activity 时很有用处。

## 添加用户界面
&emsp;&emsp;Fragment通常用作 Activity 用户界面的一部分，将其自己的布局融入 Activity。  
&emsp;&emsp;要想为Fragment提供布局，您必须实现 onCreateView() 回调方法，Android 系统会在Fragment需要绘制其布局时调用该方法。您对此方法的实现返回的 View 
必须是Fragment布局的根视图。
> **注**：如果您的Fragment是 ListFragment 的子类，则默认实现会从 onCreateView() 返回一个 ListView，因此您无需实现它。  

要想从 onCreateView() 返回布局，您可以通过 XML 中定义的布局资源来加载布局。为帮助您执行此操作，onCreateView() 提供了一个 LayoutInflater 对象。  

例如，以下这个 Fragment 子类从 example_fragment.xml 文件加载布局：
```
public static class ExampleFragment extends Fragment {
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        // Inflate the layout for this fragment
        return inflater.inflate(R.layout.example_fragment, container, false);
    }
}
```
传递至 onCreateView() 的 container 参数是您的Fragment布局将插入到的父 ViewGroup（来自 Activity 的布局）。savedInstanceState 参数是在恢复Fragment
时，提供先前的Fragment实例相关数据的 Bundle（处理片段生命周期部分对恢复状态做了详细阐述）。  
****************
inflate()方法带有三个参数：
* 想要加载的布局的资源ID
* 如果第三个参数为true，加载的布局将加入到ViewGroup(container)，container将作为加载的布局的父布局，且inflate()方法将返回container。如果第三个参数为
false，第一个参数指定的布局文件将被加载，并作为inflate()的返回值。传递 container 对系统为加载的布局的根视图设定布局参数具有重要意义，不论第三个参数是
true还是false，如果container为null，那么加载的布局的根视图的布局参数layout_width和layout_height都将作为wrap_content来处理（个人测试所得）。
* 指示加载的布局是否应该加入到ViewGroup(第二个参数)。在本例中，其值为 false，因为系统已经将加载的布局插入 container — 传递 true 值会在最终布局中创建
一个多余的view group。

## 向Activity添加Fragment
* **在 Activity 的布局文件内声明Fragment**  
在本例中，您可以将Fragment当作视图来为其指定布局属性。 例如，以下是一个具有两个Fragment的 Activity 的布局文件：
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="horizontal"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <fragment android:name="com.example.news.ArticleListFragment"
            android:id="@+id/list"
            android:layout_weight="1"
            android:layout_width="0dp"
            android:layout_height="match_parent" />
    <fragment android:name="com.example.news.ArticleReaderFragment"
            android:id="@+id/viewer"
            android:layout_weight="2"
            android:layout_width="0dp"
            android:layout_height="match_parent" />
</LinearLayout>
```
\<fragment\> 中的 android:name 属性指定要在布局中实例化的 Fragment 类。  
当系统创建此 Activity 布局时，会实例化在布局中指定的每个Fragment，并为每个Fragment调用 onCreateView() 方法，以检索每个Fragment的布局。系统会直接
插入Fragment返回的 View 来替代 \<fragment\> 元素。
> **注**：每个Fragment都需要一个唯一的标识符，重启 Activity 时，系统可以使用该标识符来恢复片段（您也可以使用该标识符来获得Fragment以执行某些事务，如将其移除）。
有两种方式为Fragment提供 ID：
1. 为 android:id 属性提供唯一 ID。
2. 为 android:tag 属性提供唯一字符串。

* **或者通过编程方式将Fragment添加到某个存在的 ViewGroup**  
您可以在 Activity 运行期间随时将Fragment添加到 Activity 布局中。您只需指定要将Fragment放入哪个 ViewGroup。
要想在您的 Activity 中执行Fragment事务（如添加、移除或替换Fragment），您必须使用 FragmentTransaction 中的 API。您可以像下面这样从 Activity 获取
一个 FragmentTransaction 实例：
```
FragmentManager fragmentManager = getFragmentManager();
FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
```
然后，您可以使用 add() 方法添加一个片段，指定要添加的片段以及将其插入哪个视图。例如：
```
ExampleFragment fragment = new ExampleFragment();
fragmentTransaction.add(R.id.fragment_container, fragment);
fragmentTransaction.commit();
```
传递到 add() 的第一个参数是 ViewGroup，即应该放置片段的位置，由资源 ID 指定，第二个参数是要添加的片段。

一旦您通过 FragmentTransaction 做出了更改，就必须调用 commit() 以使更改生效。

* 添加没有 UI 的Fragment
还可以使用Fragment为 Activity 提供后台行为，而不显示额外 UI
