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
false，第一个参数指定的布局文件将被加载，并作为inflate()的返回值，但加载的视图并不会插入container中，而需要自己调用父布局的addView手动插入。传递 container 对系统为加载的布局的根视图设定布局参数具有重要意义，不论第三个参数是
true还是false，如果container为null，那么加载的视图的布局参数LayoutParams，例如layout_width和layout_height都将使用默认参数，因为系统解析该xml文件时并不知道它的父布局是什么类型，所以就不会为它设定LayoutParams，即使它设置了layout_width等属性。但可以自己为视图调用setLayoutParams来主动设定布局参数，而避免没有传入container的情况下使用了默认参数。
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
有三种方式为Fragment提供 ID：
1. 为 android:id 属性提供唯一 ID。
2. 为 android:tag 属性提供唯一字符串。
3. 如果以上两个属性均未指定，则使用FragmentTransaction的add()方法时，第一个参数指定的Fragment的容器ID将作为Fragment的ID。（如果使用同一容器ID调用add()方法添加多个Fragment，则使用findFragmentById(容器ID)时将返回最后一个添加的Fragment实例）。

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

要想添加没有 UI 的Fragment，请从 Activity 使用FragmentTransaction类的 add(Fragment, String) 方法添加Fragment（为Fragment提供一个唯一的字符串“tag”，而不是视图 ID）。这会添加Fragment，但由于它并不与 Activity 布局中的视图关联，因此不会收到对 onCreateView() 的调用。因此，您不需要实现该方法。

并非只能为非 UI Fragment提供字符串标记 — 您也可以为具有 UI 的Fragment提供字符串标记 — 但如果片段没有 UI，则字符串标记将是标识它的唯一方式。如果您想稍后从 Activity 中获取片段，则需要使用 findFragmentByTag()。

## 管理Fragment
要想管理您的 Activity 中的Fragment，您需要使用 FragmentManager。要想获取它，请从您的 Activity 调用 getFragmentManager()。  

可以使用 FragmentManager 执行的部分操作包括：
* 通过 findFragmentById()（对于在 Activity 布局中提供 UI 的Fragment）或 findFragmentByTag()（对于提供或不提供 UI 的Fragment）获取 Activity 中存在的片段。
* 通过 popBackStack()（模拟用户发出的返回命令）将Fragment从返回栈中弹出。
* 通过 addOnBackStackChangedListener() 注册一个侦听返回栈变化的侦听器。

也可以使用 FragmentManager 打开一个 FragmentTransaction，通过它来执行某些事务，如添加和移除片段。

## 执行Fragment事务
&emsp;&emsp;在 Activity 中使用片段的一大优点是，可以根据用户行为通过它们执行添加、移除、替换以及其他操作。 您提交给 Activity 的每组更改都称为事务，您可以使用 FragmentTransaction 中的 API 来执行一项事务。您也可以将每个事务保存到由 Activity 管理的返回栈内，从而让用户能够回退片段更改（类似于回退 Activity）。

可以像下面这样从 FragmentManager 获取一个 FragmentTransaction 实例：
```
FragmentManager fragmentManager = getFragmentManager();
FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
```
每个事务都是您想要在同一时间执行的一组更改。您可以使用 add()、remove() 和 replace() 等方法为给定事务设置您想要执行的所有更改。然后，要想将事务应用到 Activity，您必须调用 commit()。detach()会执行到onPause()-->onStop()-->onDestroyView()，不会执行到onDestroy()，onDetach()。

不过，在您调用 commit() 之前，您可能想调用 addToBackStack()，以将事务添加到片段事务返回栈。 该返回栈由 Activity 管理，允许用户通过按返回按钮返回上一片段状态。

例如，以下示例说明了如何将一个片段替换成另一个片段，以及如何在返回栈中保留先前状态：
```
// Create new fragment and transaction
Fragment newFragment = new ExampleFragment();
FragmentTransaction transaction = getFragmentManager().beginTransaction();

// Replace whatever is in the fragment_container view with this fragment,
// and add the transaction to the back stack
transaction.replace(R.id.fragment_container, newFragment);
transaction.addToBackStack(null);

// Commit the transaction
transaction.commit();
```
在上例中，newFragment 会替换目前在 R.id.fragment_container ID 所标识的布局容器中的任何Fragment（如有）。通过调用 addToBackStack() 可将替换事务保存到返回栈，以便用户能够通过按返回按钮撤消事务并回退到上一Fragment。

如果您向事务添加了多个更改（如又一个 add() 或 remove()），并且调用了 addToBackStack()，则在调用 commit() 前应用的所有更改都将作为单一事务添加到返回栈，并且返回按钮会将它们一并撤消。

向 FragmentTransaction 添加更改的顺序无关紧要，不过：
* 您必须最后调用 commit()
* 如果您要向同一容器添加多个Fragment，则您添加片段的顺序将决定它们在视图层次结构中的出现顺序

如果您没有在执行移除Fragment的事务时调用 addToBackStack()，则事务提交时该Fragment会被销毁，用户将无法回退到该Fragment。 不过，如果您在删除Fragment时调用了 addToBackStack()，则Fragment会stopped，并在用户回退时将其resumed。

> 提示：对于每个片段事务，您都可以通过在提交前调用 setTransition() 来应用过渡动画。

调用 commit() 不会立即执行事务，而是在 Activity 的 UI 线程（“主”线程）可以执行该操作时再安排其在线程上运行。不过，如有必要，您也可以从 UI 线程调用 executePendingTransactions() 以立即执行 commit() 提交的事务。通常不必这样做，除非其他线程中的作业依赖该事务。

> **注意**：您只能在 Activity 保存其状态（用户离开 Activity）之前使用 commit() 提交事务。如果您试图在该时间点后提交，则会引发异常。 这是因为如需恢复 Activity，则提交后的状态可能会丢失。 对于丢失提交无关紧要的情况，请使用 commitAllowingStateLoss()。

## 与Activity通信
尽管 Fragment 是作为独立于 Activity 的对象实现，并且可在多个 Activity 内使用，但片段的给定实例会直接绑定到包含它的 Activity。

具体地说，片段可以通过 getActivity() 访问 Activity 实例，并轻松地执行在 Activity 布局中查找视图等任务。
```
View listView = getActivity().findViewById(R.id.list);
```
同样地，您的 Activity 也可以使用 findFragmentById() 或 findFragmentByTag()，通过从 FragmentManager 获取对 Fragment 的引用来调用Fragment中的方法。例如：
```
ExampleFragment fragment = (ExampleFragment) getFragmentManager().findFragmentById(R.id.example_fragment);
```

### 创建对Activity的事件回调
在某些情况下，需要通过Fragment与Activity共享事件（还是Fragment中与Activity通信）。可以在Fragment内定义一个回调接口，并要求Activity实现它。  
示例代码可参考[官网中Fragment的事件回调部分](https://developer.android.google.cn/guide/components/fragments.html#CommunicatingWithActivity)

## 向应用栏（App Bar）中添加item
Fragment可以通过实现 onCreateOptionsMenu() 向 Activity 的[选项菜单](https://developer.android.google.cn/guide/topics/ui/menus.html#options-menu)（并因此向App Bar）贡献菜单项。

&emsp;&emsp;不过，为了使此方法能够收到调用，您必须在 Fragment 的 onCreate() 期间调用 setHasOptionsMenu()，以指示Fragment想要向选项菜单添加菜单项（否则，Fragment将不会收到对 onCreateOptionsMenu() 的调用）。

&emsp;&emsp;从Fragment添加到选项菜单的任何菜单项都将追加到现有菜单项之后。 选定菜单项时，Fragment还会收到对 onOptionsItemSelected() 的回调。

&emsp;&emsp;您还可以通过调用 registerForContextMenu()，在Fragment布局中注册一个视图来提供上下文菜单。用户打开上下文菜单时，Fragment会收到对 onCreateContextMenu() 的调用。当用户选择某个菜单项时，Fragment会收到对 onContextItemSelected() 的调用。
> **注**：尽管 Fragment 会收到与其添加的每个菜单项对应的菜单项选定回调，但当用户选择菜单项时，Activity 会首先收到相应的回调。 如果 Activity 对on-item-selected 的回调的实现不处理选定的菜单项，则系统会将事件传递到Fragment的回调。 这适用于选项菜单和上下文菜单。

关于菜单的更多信息，参考[菜单](https://developer.android.google.cn/guide/topics/ui/menus.html)指南和[应用栏\(App Bar\)](https://developer.android.google.cn/training/appbar/index.html)Training。

## 处理Fragment的生命周期
管理片段生命周期与管理 Activity 生命周期很相似。和 Activity 一样，片段也以三种状态存在：  
**Resumed**  
Fragment在运行中的 Activity 中可见。
**Paused**  
另一个 Activity 位于前台并具有焦点，但此片段所在的 Activity 仍然可见（前台 Activity 部分透明，或未覆盖整个屏幕）。
**Stopped**  
Fragment不可见。要么宿主 Activity 已停止，要么Fragment已从 Activity 中移除，但已被添加到返回栈。 停止Fragment仍然处于活动状态（系统会保留所有状态和成员信息）。 不过，它对用户不再可见，而且如果 Activity 被杀死，它也会被杀死。  
![官方Fragment生命周期图](https://developer.android.google.cn/images/fragment_lifecycle.png)
&emsp;&emsp;与 Activity 一样，假使 Activity 的进程被杀死，而您需要在重建 Activity 时恢复Fragment状态，您也可以使用 Bundle 保留Fragment的状态。您可以在Fragment的 onSaveInstanceState() 回调期间保存状态，并可在 onCreate()、onCreateView() 或 onActivityCreated() 期间恢复状态。  

&emsp;&emsp;Activity 生命周期与Fragment生命周期之间的最显著差异在于它们在其各自返回栈中的存储方式。 默认情况下，Activity 停止时会被放入由系统管理的 Activity 返回栈（以便用户通过返回按钮回退到 Activity，任务和返回栈对此做了阐述）。不过，只有当您在移除Fragment的事务执行期间通过调用 addToBackStack() 显式请求保存实例时，系统才会将Fragment放入由宿主 Activity 管理的返回栈。  
> **注意**：如需 Fragment 内的某个 Context 对象，可以调用 getActivity()。但要注意，请仅在Fragment附加到 Activity 时调用 getActivity()。如果Fragment尚未附加，或在其生命周期结束期间分离，则 getActivity() 将返回 null。

## Fragment的状态保存与恢复
**保存**源码分析： 在FragmentActivity的onSaveInstanceState()方法中：
```
protected void onSaveInstanceState(Bundle outState) {
    super.onSaveInstanceState(outState);
    markFragmentsCreated();
    Parcelable p = mFragments.saveAllState();
    if (p != null) {
        outState.putParcelable(FRAGMENTS_TAG, p);
    }
    ......省略后面
    }
```
mFragments.saveAllState()会执行到FragmentManagerImpl.saveAllState()：
```
    Parcelable saveAllState() {
        ......
        //mActive为存储Fragment的键值对
        if (mActive == null || mActive.size() <= 0) {
            return null;
        }

        // First collect all active fragments.
        int N = mActive.size();
        //创建FragmentState数组
        FragmentState[] active = new FragmentState[N];
        boolean haveFragments = false;
        for (int i=0; i<N; i++) {
            Fragment f = mActive.valueAt(i);  //遍历每个Fragment
            if (f != null) {
                ......
                FragmentState fs = new FragmentState(f);  
                active[i] = fs;  //创建FragmentState对象，存在前面创建的数组中，FragmentState中包含了Bundle mArguments
                                 //和Bundle mSavedFragmentState等数据。mArguments用于Fragment.setArguments()和
                                 //Fragment.setArguments()，mSavedFragmentState用于onSaveInstanceState()和on...。

                if (f.mState > Fragment.INITIALIZING && fs.mSavedFragmentState == null) {
                    fs.mSavedFragmentState = saveFragmentBasicState(f);  //设置Bundle mSavedFragmentState，调用了saveFragmentBasicState()

                    if (f.mTarget != null) {
                        ......
                        if (fs.mSavedFragmentState == null) {
                            fs.mSavedFragmentState = new Bundle();
                        }
                        putFragment(fs.mSavedFragmentState,
                                FragmentManagerImpl.TARGET_STATE_TAG, f.mTarget);
                        if (f.mTargetRequestCode != 0) {
                            fs.mSavedFragmentState.putInt(
                                    FragmentManagerImpl.TARGET_REQUEST_CODE_STATE_TAG,
                                    f.mTargetRequestCode);
                        }
                    }

                } else {
                    fs.mSavedFragmentState = f.mSavedFragmentState;
                }
            }
        }

        if (!haveFragments) {
            if (DEBUG) Log.v(TAG, "saveAllState: no fragments!");
            return null;
        }
        ......
        FragmentManagerState fms = new FragmentManagerState();
        fms.mActive = active;  //将active数组存入FragmentManagerState对象，返回。active数组中包含了每个Fragment的mArgument                                                  //和mSavedFragmentState
        fms.mAdded = added;  //保存已经add的Fragment
        fms.mBackStack = backStack;  //保存返回栈
        if (mPrimaryNav != null) {
            fms.mPrimaryNavActiveIndex = mPrimaryNav.mIndex;
        }
        fms.mNextFragmentIndex = mNextFragmentIndex;
        saveNonConfig();
        return fms;  //返回的值将保存到Activity的Bundle中
    }
```
在saveFragmentBasicState()中：
```
    Bundle saveFragmentBasicState(Fragment f) {
        Bundle result = null;

        if (mStateBundle == null) {
            mStateBundle = new Bundle();
        }
        //其中会调用Fragment的onSaveInstanceState(Bundle outState)方法
        f.performSaveInstanceState(mStateBundle);
        dispatchOnFragmentSaveInstanceState(f, mStateBundle, false);
        if (!mStateBundle.isEmpty()) {
            // 确实有保存的东西则设置result
            result = mStateBundle;
            mStateBundle = null;
        }
        // 保存fragment view树的状态
        if (f.mView != null) {
            saveFragmentViewState(f);
        }
        if (f.mSavedViewState != null) {
            if (result == null) {
                result = new Bundle();
            }
            // 将view树的状态作为一个key、value放到result中
            result.putSparseParcelableArray(
                    FragmentManagerImpl.VIEW_STATE_TAG, f.mSavedViewState);
        }
        if (!f.mUserVisibleHint) {
            if (result == null) {
                result = new Bundle();
            }
            // Only add this if it's not the default value
            result.putBoolean(FragmentManagerImpl.USER_VISIBLE_HINT_TAG, f.mUserVisibleHint);
        }

        return result;
    }
```
**恢复**源码分析：在FragmentActivity的onCreate(Bundle savedInstanceState)中调用了mFragments.restoreAllState方法（onCreate最后会调用mFragments.dispatchCreate()方法，重走Fragment生命周期），最终调用到FragmentManagerImpl的restoreAllState方法。
```
    void restoreAllState(Parcelable state, FragmentManagerNonConfig nonConfig) {
        ......
        //先获取到保存的FragmentManagerState对象
        FragmentManagerState fms = (FragmentManagerState)state;
        if (fms.mActive == null) return;

        List<FragmentManagerNonConfig> childNonConfigs = null;
        List<ViewModelStore> viewModelStores = null;

        // First re-attach any non-config instances we are retaining back
        // to their saved state, so we don't try to instantiate them again.
        if (nonConfig != null) {
            List<Fragment> nonConfigFragments = nonConfig.getFragments();
            childNonConfigs = nonConfig.getChildNonConfigs();
            viewModelStores = nonConfig.getViewModelStores();
            final int count = nonConfigFragments != null ? nonConfigFragments.size() : 0;
            for (int i = 0; i < count; i++) {
                Fragment f = nonConfigFragments.get(i);
                if (DEBUG) Log.v(TAG, "restoreAllState: re-attaching retained " + f);
                int index = 0; // index into fms.mActive
                while (index < fms.mActive.length && fms.mActive[index].mIndex != f.mIndex) {
                    index++;
                }
                if (index == fms.mActive.length) {
                    throwException(new IllegalStateException("Could not find active fragment "
                            + "with index " + f.mIndex));
                }
                FragmentState fs = fms.mActive[index];
                fs.mInstance = f;
                f.mSavedViewState = null;
                f.mBackStackNesting = 0;
                f.mInLayout = false;
                f.mAdded = false;
                f.mTarget = null;
                if (fs.mSavedFragmentState != null) {
                    fs.mSavedFragmentState.setClassLoader(mHost.getContext().getClassLoader());
                    f.mSavedViewState = fs.mSavedFragmentState.getSparseParcelableArray(
                            FragmentManagerImpl.VIEW_STATE_TAG);
                    f.mSavedFragmentState = fs.mSavedFragmentState;
                }
            }
        }

        // Build the full list of active fragments, instantiating them from
        // their saved state.
        mActive = new SparseArray<>(fms.mActive.length);
        for (int i=0; i<fms.mActive.length; i++) {
            FragmentState fs = fms.mActive[i];
            if (fs != null) {
                FragmentManagerNonConfig childNonConfig = null;
                if (childNonConfigs != null && i < childNonConfigs.size()) {
                    childNonConfig = childNonConfigs.get(i);
                }
                ViewModelStore viewModelStore = null;
                if (viewModelStores != null && i < viewModelStores.size()) {
                    viewModelStore = viewModelStores.get(i);
                }
                Fragment f = fs.instantiate(mHost, mContainer, mParent, childNonConfig,
                        viewModelStore);  //实例化Fragment
                if (DEBUG) Log.v(TAG, "restoreAllState: active #" + i + ": " + f);
                mActive.put(f.mIndex, f);
                // Now that the fragment is instantiated (or came from being
                // retained above), clear mInstance in case we end up re-restoring
                // from this FragmentState again.
                fs.mInstance = null;
            }
        }

        // Update the target of all retained fragments.
        if (nonConfig != null) {
            ......
        }

        // Build the list of currently added fragments.
        mAdded.clear();
        if (fms.mAdded != null) {
            ......
        }

        // Build the back stack.
        if (fms.mBackStack != null) {
            ......
        } else {
            mBackStack = null;
        }

        if (fms.mPrimaryNavActiveIndex >= 0) {
            mPrimaryNav = mActive.get(fms.mPrimaryNavActiveIndex);
        }
        this.mNextFragmentIndex = fms.mNextFragmentIndex;
    }
```
Activity会自动重建Fragment，Fragment会重走完整生命周期。如果Fragment调用了setRetainInstance，则只会onDestroyView()和onDetach()，不会onDestroy()，重建时调用onAttach()和onActivityCreated()，不会调用onCreate()。    
因为重建后的Fragment是一个新的对象，可以通过FragmentManager的putFragment()和getFragment()方法配合使用，用于获取重建的那个Fragment对象。
### 与 Activity 生命周期协调
![Activity生命周期对Fragment生命周期的影响](https://developer.android.google.cn/images/activity_fragment_lifecycle.png)  
Fragment所在的 Activity 的生命周期会直接影响Fragment的生命周期，其表现为，Activity 的每次生命周期回调都会引发每个Fragment的类似回调。 例如，当 Activity 收到 onPause() 时，Activity 中的每个Fragment也会收到 onPause()。  

不过，Fragment还有几个额外的生命周期回调，用于处理与 Activity 的唯一交互，以执行构建和销毁Fragment UI 等操作。 这些额外的回调方法是：
* onAttach() 在片段已与 Activity 关联时调用（Activity 传递到此方法内）。
* onCreateView() 调用它可创建与片段关联的视图层次结构。
* onActivityCreated() 在 Activity 的 onCreate() 方法已返回时调用。
* onDestroyView() 在移除与片段关联的视图层次结构时调用。
* onDetach() 在取消片段与 Activity 的关联时调用。  

上图所示说明了受其宿主 Activity 影响的Fragment生命周期流。在该图中，您可以看到 Activity 的每个连续状态如何决定Fragment可以收到的回调方法。 例如，当 Activity 收到其 onCreate() 回调时，Activity 中的Fragment只会收到 onActivityCreated() 回调。  

一旦 Activity 达到Resumed状态，您就可以随意向 Activity 添加Fragment和移除其中的Fragment。 因此，只有当 Activity 处于Resumed状态时，Fragment的生命周期才能独立变化。  

不过，当 Activity 离开Resumed状态时，Fragment会在 Activity 的推动下再次经历其生命周期。

[官方Fragment示例](https://developer.android.google.cn/guide/components/fragments.html#Example)
