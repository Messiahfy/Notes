# Styles and Themes（样式和主题）
Android上的样式和主题允许您将应用程序设计的细节与UI结构和行为分开，类似于网页设计中的样式表。  

style样式是指定单个view外观的属性集合。 样式可以指定属性，如字体颜色，字体大小，背景颜色等等。  

theme主题是一种适用于整个应用程序、activity或view层次结构的样式，而不仅仅是单个view。 当您将样式应用为主题时，应用或活动中的每个视图都会应用它支持的每
个样式属性。主题还可以将样式应用于非视图(non-view)元素，例如状态栏和窗口背景。

样式和主题在res/values/中的style资源文件中声明，通常命名为styles.xml。  

## 创建和应用样式
要创建新样式或主题，请打开项目的res/values/styles.xml文件。 对于您要创建的每种样式，请按照下列步骤操作：

1. 添加一个具有唯一标识样式名称的<style>元素。
2. 为要定义的每个style属性添加一个<item>元素。

每个item中的name指定一个属性，对应布局中的XML属性。 <item>元素中的值是该属性的值。

例如，如果你定义如下style：
```
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <style name="GreenText" parent="TextAppearance.AppCompat">
        <item name="android:textColor">#00FF00</item>
    </style>
</resources>
```
你可以应用该style应用于一个view，如下：
```
<TextView
    style="@style/GreenText"
    ... />
```

在style中指定的每个属性都应用于该view，只要该属性能被view接受。 该view简单地忽略它不接受的任何属性。
> **注意**：只有添加style属性的元素才会接收这些样式属性----任何子视图都不会应用样式。 如果您希望子视图继承样式，请将样式应用于android：theme属性。  

但是，一般不将样式应用于单个视图，而是通常将样式应用为整个应用程序，活动或视图集合的主题。

## 扩展和自定义样式
创建自己的样式时，应始终从框架或支持库中扩展现有样式，以便保持与平台UI样式的兼容性。 要扩展样式，请使用parent属性指定要扩展的样式。 然后，您可以覆盖
继承的样式属性并添加新的属性。

例如，您可以继承Android平台的默认文本外观并对其进行如下修改：
```
<style name="GreenText" parent="@android:style/TextAppearance">
    <item name="android:textColor">#00FF00</item>
</style>
```
但是，您应该始终从Android支持库继承您的核心应用程序样式。 支持库中的样式通过针对每个版本中可用的UI属性优化每种样式，提供了与Android 4.0（API级别14）
和更高级别的兼容性。 支持库样式通常具有与来自平台的样式相似的名称，但包含AppCompat。

要从库或自己的项目中继承样式，请声明父样式名称，而不要使用上面显示的@android:style/部分。 例如，以下示例从支持库继承文本外观样式：
```
<style name="GreenText" parent="TextAppearance.AppCompat">
    <item name="android:textColor">#00FF00</item>
</style>
```

您还可以通过使用**点符号扩展样式**名称而不是使用父属性来继承样式（**除了来自平台的样式**）。 也就是说，将样式的名称加上想要继承的样式的名称，用句点分隔。 通常
只有在扩展自己的样式时才会这样做，而不是其他库中的样式。 例如，以下样式继承了上述GreenText样式中的所有样式，然后增加文本大小：
```
<style name="GreenText.Large">
    <item name="android:textSize">22dp</item>
</style>
```
您可以继续按照您希望的方式多次继承这种样式，方法是链接更多名称。

> **注意**：如果使用点符号来扩展样式，并且还包含parent属性，则父类样式会覆盖通过点符号继承的所有样式。

要找到可以用<item>标签声明哪些属性，请参阅各种类参考中的“XML attribute”表。 所有视图都支持基本View类中的XML属性，许多视图添加了它们自己的特殊属性。 例如，
TextView XML属性包含android：inputType属性，您可以将其应用于接收输入的文本视图，如EditText小部件

## 将样式应用为主题
您可以使用创建样式的相同方式创建主题。 区别在于如何应用它：不是在视图上应用具有style属性的样式，而是在AndroidManifest.xml文件的<application>标记或
<activity>标记上应用具有android：theme属性的主题。

例如，以下是如何将Android支持库的material design "dark" 主题应用于整个应用程序：
```
<manifest ... >
    <application android:theme="@style/Theme.AppCompat" ... >
    </application>
</manifest>
```
以下是如何将“light”主题应用于一项Activity：
```
<manifest ... >
    <application ... >
        <activity android:theme="@style/Theme.AppCompat.Light" ... >
        </activity>
    </application>
</manifest>
```

现在，应用或活动中的每个视图都会应用给定主题中定义的样式。 如果视图仅支持样式中声明的某些属性，则它仅应用这些属性并忽略它不支持的属性。

从Android 5.0（API级别21）和Android Support Library v22.1开始，您还可以在布局文件中将android：theme属性指定给视图。 这会修改该视图和任何子视图的
主题，这对于更改界面的特定部分中的主题调色板（theme color palettes）很有用。

前面的例子展示了如何应用Android支持库提供的主题，如Theme.AppCompat。 但您通常需要自定义主题以适应您的应用的品牌。 这样做的最好方法是从支持库扩展这些
样式，并覆盖一些属性，如下一节所述。

### 自定义默认主题
当您使用Android Studio创建项目时，默认情况下它将 material design 主题应用于您的应用，如您项目的styles.xml文件中所定义。 此AppTheme样式从支持库中
扩展了一个主题，并包括关键UI元素（如app bar和floating action button（如果使用））使用的颜色属性的覆盖。 因此，您可以通过更新提供的颜色来快速定制应
用程序的颜色设计。

例如，您的styles.xml文件应该看起来类似于以下内容：
```
<style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
    <!-- Customize your theme here. -->
    <item name="colorPrimary">@color/colorPrimary</item>
    <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
    <item name="colorAccent">@color/colorAccent</item>
</style>
```
请注意，样式值实际上是对项目res/values/colors.xml文件中定义的其他颜色资源的引用。 所以这是您应该编辑以更改颜色的文件。 但在开始更改这些颜色之前，请
使用[Material Color Tool](https://material.io/color/#!/?view.left=0&view.right=0)预览颜色。 此工具可帮助您从素材调色板中选取颜色并预览它们在应
用中的外观。

一旦你知道你的颜色，更新res / values / colors.xml中的值：
```
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <!--   color for the app bar and other primary UI elements -->
    <color name="colorPrimary">#3F51B5</color>

    <!--   a darker variant of the primary color, used for
           the status bar (on Android 5.0+) and contextual app bars -->
    <color name="colorPrimaryDark">#303F9F</color>

    <!--   a secondary color for controls like checkboxes and text fields -->
    <color name="colorAccent">#FF4081</color>
</resources>
```
然后你可以重写你想要的任何其他样式。 例如，您可以按如下方式更改活动背景颜色：
```
<style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
    ...
    <item name="android:windowBackground">@color/activityBackground</item>
</style>
```
有关可用于主题的属性列表，请参阅[R.styleable.Theme](https://developer.android.google.cn/reference/android/R.styleable.html#Theme)上的属性表。
当为布局中的视图添加样式时，您还可以通过查看视图类引用中的“XML attribute”表来查找属性。 例如，所有视图都支持基本View类的XML属性。

大多数属性应用于特定类型的视图，并且一些属性适用于所有视图。 然而，在[R.styleable.Theme](https://developer.android.google.cn/reference/android/R.styleable.html#Theme)
列出的一些主题属性适用于活动窗口，而不是布局中的视图。 例如，windowBackground更改窗口背景，windowEnterTransition定义要在活动开始时使用的过渡动画。

Android支持库还提供了其他可用于自定义从Theme.AppCompat扩展的主题（如上面显示的colorPrimary属性）的属性。 这些最好在[library's attrs.xml](https://android.googlesource.com/platform/frameworks/support/+/master/v7/appcompat/res/values/attrs.xml)文件中查看
> 注意：支持库中的属性名称不使用android:前缀。 该前缀仅用于Android framework的属性。

支持库中还有不同的主题，您可能想要扩展而不是上面显示的主题。 查看可用主题的最佳位置是[library's themes.xml](https://android.googlesource.com/platform/frameworks/support/+/master/v7/appcompat/res/values/themes.xml)文件。

## 添加特定于版本的样式
如果新版本的Android添加了您想要使用的主题属性，则可以将它们添加到主题中，同时仍与旧版本兼容。 您所需要的只是保存在包含资源版本限定符的value目录中的
另一个styles.xml文件。 例如：

res/values/styles.xml        # themes for all versions
res/values-v21/styles.xml    # themes for API level 21+ only

由于values/styles.xml文件中的样式适用于所有版本，因此values-v21/styles.xml中的主题可以继承它们。 因此，您可以通过从“base”主题开始，然后将其扩展到
特定于版本的样式来避免重复样式。

例如，要声明Android 5.0（API级别21）及更高版本的窗口转换，您需要使用一些新的属性。 所以你的基本主题可能是这样的：
*res/values/styles.xml*
```
<resources>
    <!-- base set of styles that apply to all versions -->
    <style name="BaseAppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
        <item name="colorPrimary">@color/primaryColor</item>
        <item name="colorPrimaryDark">@color/primaryTextColor</item>
        <item name="colorAccent">@color/secondaryColor</item>
    </style>

    <!-- declare the theme name that's actually applied in the manifest file -->
    <style name="AppTheme" parent="BaseAppTheme" />
</resources>
```
然后，只需在其他文件中添加版本特定的样式，如下所示：
*res/values-v21/styles.xml*
```
<resources>
    <!-- extend the base theme to add styles available only with API level 21+ -->
    <style name="AppTheme" parent="BaseAppTheme">
        <item name="android:windowActivityTransitions">true</item>
        <item name="android:windowEnterTransition">@android:transition/slide_right</item>
        <item name="android:windowExitTransition">@android:transition/slide_left</item>
    </style>
</resources>
```
现在，您可以在清单文件中应用AppTheme，并且系统会选择可用于每个系统版本的样式。

## 自定义widget样式
framework和支持库中的每个widget都有一个默认样式。 例如，当您使用support library中的主题设计应用程序样式时，Button的一个实例使用Widget.AppCompat.Button
样式进行样式设置。 如果您想将不同的widget样式应用于按钮，则你可以使用布局文件中的style属性。 例如，以下应用库的无边框按钮样式：
```
<Button
    style="@style/Widget.AppCompat.Button.Borderless"
    ... />
```
如果您想将此样式应用于所有按钮，则可以在主题的buttonStyle中声明它，如下所示：
```
<style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
    <item name="buttonStyle">@style/Widget.AppCompat.Button.Borderless</item>
    ...
</style>
```
您还可以扩展widget样式，就像扩展任何其他样式一样，然后将自定义widget样式应用于布局或主题中。

要发现支持库中可用的所有替代小部件样式，请查看以Widget开头的字段的[R.style](https://developer.android.google.cn/reference/android/support/v7/appcompat/R.style.html)参考。 （忽略以Base_Widget开头的样式。）请记住在资源中使用样式名称时用句点替换所有下划线。
