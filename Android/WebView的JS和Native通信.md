# 概述
当前的移动端App开发，很多项目都会集成Web技术，Android中使用WebView可以展示前端Web页面，并且WebView的web页面（JavaScript）往往要和Native（Java/Kotlin）相互通信，所以需要使用相应的通信方式。

# WebView API
在了解通信方式之前，我们先了解一下WebView提供的API，对WebView提供的功能有更系统的认知。
## 基本使用
声明网络权限
```
// 一般需要声明网络权限
<uses-permission android:name="android.permission.INTERNET"/>
```

常用配置
```
// 支持JavaScript
webView.settings.javaScriptEnabled = true
//是否打开Dom本地存储，有些web页面使用了Dom Storgage（Web Storage）存储机制，这种情况需要打开
webView.settings.domStorageEnabled = true

// 设置WebViewClient，可以自定义各种事件通知请求的回调
binding.webView.webViewClient = WebViewClient()
// 设置WebChromeClient，可以自定义JS弹窗、标题、进度等
binding.webView.webChromeClient = WebChromeClient()

// 设置WebView背景色
binding.webView.setBackgroundColor(color)
```

加载url，即可完成web页面展示。url可以是http远程资源或本地文件。
```
// 加载url对应的web页面
binding.webView.loadUrl(url)
```

## 其他API介绍
1. WebView自身API
```
// 基本信息

String getUrl() // 获取当前url
String getTitle() // 获取当前标题
Bitmap getFavicon() // 获取当前favicon
int getProgress() // 加载进度
static void setWebContentsDebuggingEnabled // 调试

// 配置
WebSettings getSettings()
void setWebViewClient(@NonNull WebViewClient client) // WebViewClient可以设置多种事件回调
void setWebChromeClient(@Nullable WebChromeClient client) // WebChromeClient可以处理js弹窗、图标等

// JavaScript

void addJavascriptInterface(@NonNull Object object, @NonNull String name) // 注入JS对象
void removeJavascriptInterface(@NonNull String name)
// Android异步执行WebView当前显示页面中的JavaScript的方法。需要在UI线程调用
void evaluateJavascript(@NonNull String script, @Nullable ValueCallback<String> resultCallback)

// 加载URL

void loadUrl(@NonNull String url, @NonNull Map<String, String> additionalHttpHeaders) // 加载URL，可配置header
void loadUrl(@NonNull String url) // // 加载URL
void postUrl(@NonNull String url, @NonNull byte[] postData) // 使用POST请求
void loadData(@NonNull String data, @Nullable String mimeType, @Nullable String encoding) // 加载字符串数据，比如字符串是一个html数据
void loadDataWithBaseURL(@Nullable String baseUrl, @NonNull String data,
            @Nullable String mimeType, @Nullable String encoding, @Nullable String historyUrl)
void reload() // 重新加载当前url
void stopLoading() // 停止当前加载

// WebView网络是否可用
void setNetworkAvailable(boolean networkUp)

// 销毁WebView的内部状态，在从View树中移除后才能调用，调用此方法后不不能再调用WebView的其他方法
void destroy()

// 保存当前页面到指定文件（比如html）
void saveWebArchive(@NonNull String filename)

// 历史页面回退栈管理
boolean canGoBack()
void goBack()
boolean canGoForward()
void goForward()
boolean canGoBackOrForward(int steps)
void goBackOrForward(int steps)
void clearHistory()

// 是否启用无痕浏览
boolean isPrivateBrowsingEnabled()

// 页面滑动
boolean pageUp(boolean top)
boolean pageDown(boolean bottom)
void flingScroll(int vx, int vy)

// WebView内容ui相关
void postVisualStateCallback(long requestId, @NonNull VisualStateCallback callback)
void setInitialScale(int scaleInPercent)
void invokeZoomPicker()
HitTestResult getHitTestResult()
void zoomBy(float zoomFactor)
boolean zoomIn()
boolean zoomOut()
int getContentHeight()
int getContentWidth()

// 暂停/恢复
void onPause() // 暂停WebView动画、定位等
void onResume()
void pauseTimers() // 暂停当前Application所有的WebView的layout、parse和javascript timer
void resumeTimers()

// 。。。。。。其他省略
```

2. WebSettings API
```
// 省略getter

void setJavaScriptEnabled(boolean flag) // 启用JavaScript，默认false
void setDatabaseEnabled(boolean flag) // 启用数据库存储API，默认false
void setDomStorageEnabled(boolean flag) // 启用DOM存储API，默认false
void setGeolocationEnabled(boolean flag) // 启动定位API，默认true。需要定位权限，实现WebChromeClient#onGeolocationPermissionsShowPrompt
void setJavaScriptCanOpenWindowsAutomatically(boolean flag) // 让JavaScript可以自动打开Window，默认false
void setSafeBrowsingEnabled(boolean enabled) // 启用安全浏览
void setAllowFileAccess(boolean allow) // 是否可以访问文件，比如 file:///android_asset，API 30以以上默认false，之前默认true
void setAllowContentAccess(boolean allow) // content url是否可以访问


void setSupportZoom(boolean support) // 是否支持缩放，默认true
void setMediaPlaybackRequiresUserGesture(boolean require) // 设置播放媒体是否需要用户手势，避免自动播放，默认true
void setBuiltInZoomControls(boolean enabled)
void setDisplayZoomControls(boolean enabled)
void setAlgorithmicDarkeningAllowed(boolean allow) // 深色模式
void setLoadWithOverviewMode(boolean overview) // 是否使用overview mode加载页面，默认值 false，当页面宽度大于WebView宽度时，缩小至屏幕大小
void setUseWideViewPort(boolean use) // 设置是否支持 viewport 属性，
void setSupportMultipleWindows(boolean support) // 是否支持多窗口，默认值false
void setLayoutAlgorithm(LayoutAlgorithm l) // 布局算法，默认为LayoutAlgorithm#NARROW_COLUMNS
void setTextZoom(int textZoom) // 文字缩放百分比，默认值 100
void setStandardFontFamily(String font) // 设置标准字体名称，默认为 sans-serif
void setFixedFontFamily(String font) // 设置等宽字体，默认为 monospace
void setSansSerifFontFamily(String font)
void setSerifFontFamily(String font)
void setCursiveFontFamily(String font)
void setFantasyFontFamily(String font)
void setMinimumFontSize(int size)
void setMinimumLogicalFontSize(int size)
void setDefaultFontSize(int size)
void setDefaultFixedFontSize(int size)

void setLoadsImagesAutomatically(boolean flag) // 是否自动加载图片，默认true
void setBlockNetworkImage(boolean flag)


void setDefaultTextEncodingName(String encoding) // 设置文本编码
void setUserAgentString(@Nullable String ua) // 设置user agent
void setNeedInitialFocus(boolean flag) // 调用 WebView#requsetFocus 时，会设置焦点。默认true
void setCacheMode(@CacheMode int mode) // 设置缓存模式
void setDisabledActionModeMenuItems(@MenuItemFlags int menuItems)
```

# Web和Native通信方式
前面了解了WebView的各种API，可以看到其中几个API可以用于JS/Native的通信。
## JavaScript -> Android
首先来看JavaScript调用Java/Kotlin代码，WebView提供了JavaScript代码调用Java/Kotlin代码的多种方式。

### 1. 通过WebView的 addJavascriptInterface() 注入对象
这个是最主流的方式。注入Java对象到Webview，然后JavaScript可以通过window对象间接访问这个Java对象，达到调用Java/Kotlin代码的目的。

**Android代码**：
```
class JsBridge {
    @JavascriptInterface
    public String getString() { return "test string"; }

    @JavascriptInterface
    public String log(text: String) { Log.d("text") }
}

// 设置支持 JavaScript
webView.settings.javaScriptEnabled = true
// 添加 JsBridge 实例，在 JavaScript 中通过 window.jsBridge 可以访问到此对象
webView.addJavascriptInterface(JsBridge(), "jsBridge");
webView.loadUrl(url)
```
使用 `@JavascriptInterface` 注解的public方法可以被JavaScript访问，对象将注入到 JavaScript 的 `window` 对象中，`addJavascriptInterface`方法的第二个参数将作为 JavaScript 在 `window` 中访问该JsBridge实例的属性名称。

> `addJavascriptInterface`可以调用多次，添加多个注入的对象，但如果传入name一样，后面添加的将覆盖之前添加的。

**Web代码**：
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>Document</title>
    <script>
        function callAndroid(){
          // 将调用到Android中JsBridge对象的getString()方法，返回值赋值给 s 变量
          let s = window.jsBridge.getString();
          // 将调用到Android中JsBridge对象的log(text)方法，将参数"测试文本"传到Android代码
          window.jsBridge.log("测试文本")
        }
    </script>
</head>
<body>
<button type="button" onclick="callAndroid()">按钮</button>
</body>
</html>
```
点击按钮将调用Android中声明的`getString()`和`log(text: String)`方法。`getString()`的使用情况表明JavaScript可以调用Android中的Java/Kotlin代码，并且可以将调用Android方法的返回值传回JavaScript。`log(text: String)`的使用情况表明JavaScript可以把数据传到Android的Java/Kotlin代码中。

以上基本使用示例，可以了解到`addJavascriptInterface`注解提供了JavaScript到Java/Kotlin的方法调用，支持传参和返回值。那么下面将继续分析`addJavascriptInterface`对各种调用方式的支持情况：

#### 支持的数据类型
`@JavascriptInterface` 注解的方法，支持Java的`基本数据类型`和`String`作为参数和返回值
```
// Kotlin
@JavascriptInterface
fun showHint(a: String, b: Int, c: Double): String{
    Log.d("TAG", "showHint $a $b $c")
    retrun "测试文本"
}

// JavaScript
let x = window.androidClient.showHint("1", 2 ,3);
console.log("showHint return " + x)
```
这个例子验证了String、Int、Double作为参数和返回值，均可正常运行。

验证`数组`作为方法参数的情况：
```
// Kotlin
@JavascriptInterface
fun showHint(a: Array<String>) {
    Log.d("TAG", "showHint ${a[0]} ${a[1]}")
}

// JavaScript
window.androidClient.showHint(["One", "Two"]);
```
Kotlin代码可以正常接收到JavaScript传来的数组

验证`数组`作为方法返回值的情况：
```
// Kotlin
@JavascriptInterface
fun showHint(): Array<String> {
    return 
}

// JavaScript
window.androidClient.showHint(["One", "Two"]);
```
无法正常调用，不支持数组作为返回值

验证返回不是基本类型也不是String的其他类型：
```
// Kotlin
@JavascriptInterface
fun showHint(): CustomType {
    return CustomType()
}

// JavaScript
let x = window.androidClient.showHint();
console.log(x) // JS收到{}，也就是未设置属性的对象
```

**数据类型支持情况总结**：Java基本数据类型和String可以作为参数和返回值，基本数据类型的数组和String数组只能作为参数，如果要使用自定义类型，可以使用Json等格式的字符串类型达到目的。

#### 参数不匹配的情况
1. 类型不匹配
```
// Kotlin
@JavascriptInterface
fun showHint(a: Int, b: String) {
    Log.d("TAG", "$a $b")
}

// JavaScript
window.androidClient.showHint(1.5, 2); // Android打印 1 2

window.androidClient.showHint("1", 2);  // Android打印 0 2

window.androidClient.showHint("哈哈", {}); // Android打印 0.0 undefined
```
* showHint(1.5, 2)：打印 1  2，说明Number类型之间可以转换，非String类型可以转为String
* showHint("1", 2)：打印 0  2，说明非Number类型转为Number失败，将使用默认值
* showHint("哈哈", {})：打印 0.0  undefined，说明转String失败将使用undefined


如果声明参数类型不是基本类型也不是String类型，而是其他类型：
```
// Kotlin
@JavascriptInterface
fun showHint(a: CustomType?) {
    Log.d("TAG", "$a") // 无论JS传参是什么，只会接收到null
}
```

**参数类型不匹配的情况总结**：Java/Kotlin方法接收的JavaScript传入参数类型只能是Java的内置类型或String和它们的数组，否则param为null、或为undefined（当用String接收时）。Number类型之间可以转换，但可能丢失精度。

2. 数量不匹配
```
// Kotlin
@JavascriptInterface
fun showHint(a: Int, b: Int) {
    Log.e("TAG", "$a  $b")
}

// JavaScript
window.androidClient.showHint(1)  // 报错 "Uncaught Error: Method not found"
window.androidClient.showHint(1, 2, 3)  // 报错 "Uncaught Error: Method not found"
```
说明传入参数和声明的参数数量不匹配，都会报错找不到方法。

3. JavaScript命名参数
JavaScript命名参数调用Android中`@JavascriptInterface`注解的方法，命名参数不会生效，仍然按参数的实际顺序传参。
```
// Kotlin
@JavascriptInterface
fun showHint(a: Int, b: Int) {
    Log.e("TAG", "$a  $b")
}

// JavaScript
window.androidClient.showHint(b = 1, a = 2) // 仍然打印 1 2
```


#### 方法重载的情况
`@JavascriptInterface` 支持注解多个不同参数数量的重载方法，将根据JS的传参数量情况选择对应的方法。如果参数数量相同，参数类型不同，只会选择第一个声明的方法。所以不建议使用重载，容易产生错误。

#### 线程
`@JavascriptInterface`注解的方法会在JavaBridge线程被调用，所以执行代码需要注意，必要时切换线程

#### 异步
* `@JavascriptInterface`注解的方法对于JavaScript为同步调用，如果方法没有返回值，js得到的返回值为undefined。
* 如果要要异步回调，由于传参不支持自定义类型，所以无法传递Callback对象，只能使用字符串传递方法名称，Android中再使用`evaluateJavascript`调用JS的回调函数。
* `@JavascriptInterface`注解的方法，不能使用Kotlin suspend，因为方法会增加一个Continuation参数，导致反射找不到方法

#### addJavascriptInterface 原理简单了解
JavaScript是一门脚本语言，需要使用JSEngine来解析和执行（例如谷歌V8引擎），JSEngine在执行JS代码时，会构造出运行时环境（Context），Web场景的Context内置了全局对象window和其他一些内置对象。

并且JSEngine通常会被集成到宿主中，最出名的宿主就是浏览器。为了让JS可以访问宿主的能力，JSEngine对外提供了扩展API，允许宿主向JS运行时环境注入特有的能力（注入API），并且向宿主暴露了一些hook接口，用于实现相互通信。

WebView.addJavascriptInterface最终会调用到web引擎的C++代码中的[AddJavascriptInterface](https://android.googlesource.com/platform/external/webkit/+/refs/heads/main/Source/WebKit/android/jni/WebCoreFrameBridge.cpp) 函数，调用[bindToWindowObject](https://android.googlesource.com/platform/external/webkit/+/refs/heads/main/Source/WebCore/bindings/v8/ScriptController.cpp) 绑定到js上下文 。JS调用注入的对象就是通过JS引擎通过反射调用Java/Kotlin代码。

参考文档：https://chromium.googlesource.com/chromium/src/+/master/android_webview/docs/java-bridge.md

### 2. 通过 WebViewClient 的 shouldOverrideUrlLoading() 方法回调拦截 url
重写`shouldOverrideUrlLoading`也是JS调用Java/Kotlin通信的一种方式，但不如使用上文介绍的`addJavascriptInterface`便捷和高效，所以作为了解即可。

在JS代码中，可以触发url加载，有多种方式，下面介绍常用的两种：
```
// js代码

// 1. 可以修改 window.location 的值，设置url。schema可以自行设置，这里示例使用jsbridge
window.location = 'jsbridge://xxxx'

// 2. 也可以使用不可见的 iframe 元素
let document.createElement('iframe');
iframe.style.display = 'none';  
iframe.src = 'jsbridge://111'
document.body.appendChild(iframe);
iframe.src = 'jsbridge://222'
```

JS代码中触发了url加载，在Android中可以通过重写WebViewClient的shouldOverrideUrlLoading()方法，监听到url加载，然后从url信息中得到通信的数据。
```
// Android代码
binding.webView.webViewClient = object : WebViewClient() {

    override fun shouldOverrideUrlLoading(view: WebView, request: WebResourceRequest): Boolean {
        // 如果我们想用自己的代码逻辑拦截处理就返回true，如果还是让WebView自行处理就返回false
        if(needHandle(request)){
            return true
        }
        return super.shouldOverrideUrlLoading(view, request)
    }
}
```
WebViewClient重写`shouldOverrideUrlLoading`方法，就可以根据`WebResourceRequest`的请求信息，自行拦截处理。不过这种方式就不方便像`addJavascriptInterface()`方式一样直接返回值，而要使用WebView的`evaluateJavascript`方法调用JS代码。

> webView只能识别http、https和file之类的协议，要支持一些特殊协议也可以使用 shouldOverrideUrlLoading


### 3. 通过 WebChromeClient 的onJsAlert()、onJsConfirm()、onJsPrompt（）方法回调拦截JS对话框alert()、confirm()、prompt()消息
类似第二种方式，监听JS触发的操作，不算常规通用的通信方式。


**以上3种方式，第2种和第3种都算比较偏门的做法，而第1种方式算是最主流和便捷的，所以一般情况建议采用第1种方式**

## Android -> JavaScript：
前面介绍了JS代码主动调用Android代码的方式，本节介绍Android主动调用WebView页面中的JavaScript的几种方式。

### 1. 调用WebView的 evaluateJavascript() 执行js代码 
`evaluateJavascript()`是Android 4.4版本添加的API，可以异步执行当前web页面context中的JavaScript代码，并且可以接收执行的结果回调。
```
mWebView.evaluateJavascript("javascript:jsFunction()", new ValueCallback<String>() {
        @Override
        public void onReceiveValue(String value) {
            //value为js函数的返回值
        }
    });
```
`jsFunction()`为JavaScript中的函数名称，这里的例子中执行的JS代码仅为一个函数，实际可以执行多个语句。代码字符串开头不加 javascript: 也可以执行。

### 2. 调用WebView的loadUrl()
```
// jsCode为WebView当前页面的一段JS代码
webView.loadUrl("javascript:jsCode");
```
通过这个方式也可以执行JS代码。但`loadUrl()`不能像`evaluateJavascript()` 得到结果回调。