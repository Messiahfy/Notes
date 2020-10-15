## 依赖
```
implementation 'com.github.bumptech.glide:glide:4.11.0'
annotationProcessor 'com.github.bumptech.glide:compiler:4.11.0' //用于自定义模块
```

## 源码流程
Glide的基本使用较简单，我们以Glide的经典使用方式为入口，分析源码的执行流程：
```
Glide.with(this).load(url).into(imageView);
```

### Glide#with 创建RequestManager的过程
从with方法开始，with方法有多个重载方法，：
```
@NonNull
public static RequestManager with(@NonNull Context context) {
    return getRetriever(context).get(context);
}

@NonNull
public static RequestManager with(@NonNull Activity activity) {
    return getRetriever(activity).get(activity);
}

@NonNull
public static RequestManager with(@NonNull FragmentActivity activity) {
    return getRetriever(activity).get(activity);
}

@NonNull
public static RequestManager with(@NonNull Fragment fragment) {
    return getRetriever(fragment.getContext()).get(fragment);
}

@NonNull
public static RequestManager with(@NonNull View view) {
    return getRetriever(view.getContext()).get(view);
}
```
getRetriever方法如下：
```
  @NonNull
  private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    // ...省略判空
    return Glide.get(context).getRequestManagerRetriever();
  }
```
可以看到getRetriever中只是使用传入的Context给Glide.get(context)来初始化Glide，初始化过程会用到Builder模式。然后调用getRequestManagerRetriever()得到RequestManagerRetriever实例。

这里看到前面with方法中，都会调用RequestManagerRetriever的get方法，并且可以传入context、activity等类型，以其中的FragmentActivity为例：
```
// RequestManagerRetriever#get

@NonNull
public RequestManager get(@NonNull FragmentActivity activity) {
    if (Util.isOnBackgroundThread()) {
        return get(activity.getApplicationContext());
    } else {
        assertNotDestroyed(activity);
        FragmentManager fm = activity.getSupportFragmentManager();
        return supportFragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
}
```
通过activity得到FragmentManager，然后执行supportFragmentGet方法：
```
@NonNull
private RequestManager supportFragmentGet(
    @NonNull Context context,
    @NonNull FragmentManager fm,
    @Nullable Fragment parentHint,
    boolean isParentVisible) {
    // 创建并添加一个用于存储RequestManager的无界面fragment，RequestManager会和lifecycle关联，而lifecycle会由无界面fragment触发
    SupportRequestManagerFragment current =
        getSupportRequestManagerFragment(fm, parentHint, isParentVisible);
    RequestManager requestManager = current.getRequestManager();
    // 初始情况，无界面fragment中的requestManager为null
    if (requestManager == null) {
        Glide glide = Glide.get(context);
        // 所以会这里构造requestManager。requestManager会和lifecycle关联
        requestManager =
            factory.build(
                glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
        // 把requestManager传给无界面fragment
        current.setRequestManager(requestManager);
    }
    return requestManager;
}
```
application的context使requestManager对应的生命周期只会onStart，而activity、fragment和view都会使requestManager和一个无界面的fragment关联生命周期。

### RequestManager#load 创建RequestBuilder的过程
前面创建好了RequestManager，现在调用它的load方法，load方法有多个重载，可以加载多种资源，以String类型参数为例：
```
public RequestBuilder<Drawable> load(@Nullable String string) {
    return asDrawable().load(string);
}

// ...其他各种类型的重载可看文档或源码
```

> 除了用load方法，实际还有downloadOnly和download用于下载图片。

load方法中的asDrawable()：
```
public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class);
}

// 执行到as方法
public <ResourceType> RequestBuilder<ResourceType> as(
    @NonNull Class<ResourceType> resourceClass) {
    return new RequestBuilder<>(glide, this, resourceClass, context); //this就是RequestManager
}
```
可以看到，as方法中，创建了RequestBuilder实例，持有RequestManager，并且把Drawable.class传入，表示需要把资源解码为Drawable类型

然后执行RequestBuilder的load方法，即传入资源对象，例如String、File等。到这里，已经完成RequestBuilder的创建和设置。

### RequestBuilder#into 
into方法主要有两种，一是ImageView类型，还有自定义Target的多个重载，ImageView类型的方法也会执行到自定义Target的重载方法中，因为ImageView会被包装为Target的子类ViewTarget。
```
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    //...省略
    return into(
        // glideContext在构造RequestBuilder时初始化，它是glide实例中的包含各种配置的上下文
        glideContext.buildImageViewTarget(view, transcodeClass),
        /*targetListener=*/ null,
        requestOptions,
        Executors.mainThreadExecutor());
}
```
调用into方法的ImageView参数类型重载，也是调用自定义Target重载中。首先看如何根据ImageView创建Target：
```
// GlideContext#buildImageViewTarget

public <X> ViewTarget<ImageView, X> buildImageViewTarget(
        @NonNull ImageView imageView, @NonNull Class<X> transcodeClass) {
    // imageViewTargetFactory已在Glide构造函数中初始化，transcodeClass在前面RequestManager的as方法中传入，如Drawable.class
    return imageViewTargetFactory.buildTarget(imageView, transcodeClass);
}
```
// ImageViewTargetFactory#buildTarget
```
  public <Z> ViewTarget<ImageView, Z> buildTarget(
      @NonNull ImageView view, @NonNull Class<Z> clazz) {
    if (Bitmap.class.equals(clazz)) {
      return (ViewTarget<ImageView, Z>) new BitmapImageViewTarget(view);
    } else if (Drawable.class.isAssignableFrom(clazz)) {
      return (ViewTarget<ImageView, Z>) new DrawableImageViewTarget(view);
    } else {
      throw new IllegalArgumentException(
          "Unhandled class: " + clazz + ", try .as*(Class).transcode(ResourceTranscoder)");
    }
  }
```
假设前面设置的Drawable.class，那么这里会返回DrawableImageViewTarget，然后我们回到into方法：
```
private <Y extends Target<TranscodeType>> Y into(
        @NonNull Y target,
        @Nullable RequestListener<TranscodeType> targetListener,
        BaseRequestOptions<?> options,
        Executor callbackExecutor) {
    //...
    // 构造Request
    Request request = buildRequest(target, targetListener, options, callbackExecutor);
    //...省略：相同的target已有的相同request还未执行完，或者执行完且能使用内存缓存，则复用之前request并return

    requestManager.clear(target);
    target.setRequest(request);
    requestManager.track(target, request);

    return target;
}
```

### 创建Request
buildRequest内部调用buildRequestRecursive：
```
// RequestBuilder#buildRequestRecursive

private Request buildRequestRecursive(
        Object requestLock,
        Target<TranscodeType> target,
        @Nullable RequestListener<TranscodeType> targetListener,
        @Nullable RequestCoordinator parentCoordinator,
        TransitionOptions<?, ? super TranscodeType> transitionOptions,
        Priority priority,
        int overrideWidth,
        int overrideHeight,
        BaseRequestOptions<?> requestOptions,
        Executor callbackExecutor) {

    ErrorRequestCoordinator errorRequestCoordinator = null;
    // 如果通过RequestBuilder的error方法设置了errorBuilder，就会构造errorRequestCoordinator
    if (errorBuilder != null) {
        errorRequestCoordinator = new ErrorRequestCoordinator(requestLock, parentCoordinator);
        parentCoordinator = errorRequestCoordinator;
    }
    // 未设置错误占位的普通情况，执行buildThumbnailRequestRecursive构造Request
    Request mainRequest =
        buildThumbnailRequestRecursive(
            requestLock,
            target,
            targetListener,
            parentCoordinator,
            transitionOptions,
            priority,
            overrideWidth,
            overrideHeight,
            requestOptions,
            callbackExecutor);

    if (errorRequestCoordinator == null) {
        return mainRequest;
    }

    //... 省略如果设置了错误占位，返回errorRequest的情况
}
```
buildThumbnailRequestRecursive内部首先是考虑设置缩略图的情况，返回一个包装了缩略图请求和完整图请求的请求，缩略图先完成就先显示缩略图，完整图先完成就显示完整图，并清除缩略图请求，这里不详细分析。如果没有设置缩略图，则执行obtainRequest方法返回SingleRequest。

### Request#begin
RequestBuilder的into方法中最后几步，执行requestManager的track方法，其中会引发request的begin方法执行：
```
// SingleRequest#begin

public void begin() {
    synchronized (requestLock) {
      //...
      // 如果传入图片资源为空，则执行失败回调
      if (model == null) {
        if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
          width = overrideWidth;
          height = overrideHeight;
        }
        int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
        onLoadFailed(new GlideException("Received null model"), logLevel);
        return;
      }

      if (status == Status.RUNNING) {
        throw new IllegalArgumentException("Cannot restart a running request");
      }

      // 如果已完成，说明是复用之前的request，则直接回调onResourceReady，其中就会回调各种监听，以及执行Target的onResourceReady
      if (status == Status.COMPLETE) {
        onResourceReady(resource, DataSource.MEMORY_CACHE);
        return;
      }

      status = Status.WAITING_FOR_SIZE;
      // 如果尺寸已经准备好，则直接调用onSizeReady方法
      if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        onSizeReady(overrideWidth, overrideHeight);
      } else {
        // 否则通过ViewTreeObserver的OnPreDrawListener等待view的尺寸测量完成，完成后同样调用onSizeReady方法
        target.getSize(this);
      }

      if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
          && canNotifyStatusChanged()) {
        // 先显示占位图
        target.onLoadStarted(getPlaceholderDrawable());
      }
      // ...
    }
}
```
后续步骤位于onSizeReady方法中，我们继续阅读：
```
public void onSizeReady(int width, int height) {
    stateVerifier.throwIfRecycled();
    synchronized (requestLock) {
      //...
      if (status != Status.WAITING_FOR_SIZE) {
        return;
      }
      status = Status.RUNNING;

      float sizeMultiplier = requestOptions.getSizeMultiplier();
      this.width = maybeApplySizeMultiplier(width, sizeMultiplier);
      this.height = maybeApplySizeMultiplier(height, sizeMultiplier);
      //...
      loadStatus =
          engine.load(
              glideContext,
              model,
              requestOptions.getSignature(),
              this.width,
              this.height,
              requestOptions.getResourceClass(),
              transcodeClass,
              priority,
              requestOptions.getDiskCacheStrategy(),
              requestOptions.getTransformations(),
              requestOptions.isTransformationRequired(),
              requestOptions.isScaleOnlyOrNoTransform(),
              requestOptions.getOptions(),
              requestOptions.isMemoryCacheable(),
              requestOptions.getUseUnlimitedSourceGeneratorsPool(),
              requestOptions.getUseAnimationPool(),
              requestOptions.getOnlyRetrieveFromCache(),
              this, //this为SingleRequest，执行结束要回调成功onResourceReady或失败onLoadFailed
              callbackExecutor);

      // This is a hack that's only useful for testing right now where loads complete synchronously
      // even though under any executor running on any thread but the main thread, the load would
      // have completed asynchronously.
      if (status != Status.RUNNING) {
        loadStatus = null;
      }
      //...
    }
}
```
### Engine#load
这个engine是在构造glide时初始化的，关键就是执行engine的load方法：
```
public <R> LoadStatus load(
    GlideContext glideContext,
    Object model,
    Key signature,
    int width,
    int height,
    Class<?> resourceClass,
    Class<R> transcodeClass,
    Priority priority,
    DiskCacheStrategy diskCacheStrategy,
    Map<Class<?>, Transformation<?>> transformations,
    boolean isTransformationRequired,
    boolean isScaleOnlyOrNoTransform,
    Options options,
    boolean isMemoryCacheable,
    boolean useUnlimitedSourceExecutorPool,
    boolean useAnimationPool,
    boolean onlyRetrieveFromCache,
    ResourceCallback cb, // SingleRequest实现了此接口
    Executor callbackExecutor) {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;
    // 首先构造EngineKey，只是一个数据结构，根据相关信息，作为资源缓存标识
    EngineKey key =
        keyFactory.buildKey(
            model,
            signature,
            width,
            height,
            transformations,
            resourceClass,
            transcodeClass,
            options);

    EngineResource<?> memoryResource;
    synchronized (this) {
      // 根据key，从内存中寻找缓存
      memoryResource = loadFromMemory(key, isMemoryCacheable, startTime);

      if (memoryResource == null) {
        return waitForExistingOrStartNewJob(
            glideContext,
            model,
            signature,
            width,
            height,
            resourceClass,
            transcodeClass,
            priority,
            diskCacheStrategy,
            transformations,
            isTransformationRequired,
            isScaleOnlyOrNoTransform,
            options,
            isMemoryCacheable,
            useUnlimitedSourceExecutorPool,
            useAnimationPool,
            onlyRetrieveFromCache,
            cb,
            callbackExecutor,
            key,
            startTime);
      }
    }

    // memoryResource不为null，则回调成功
    cb.onResourceReady(memoryResource, DataSource.MEMORY_CACHE);
    return null;
}
```
### 构造EngineJob和DecodeJob
从没有缓存的情况，继续查看waitForExistingOrStartNewJob：
```
  private <R> LoadStatus waitForExistingOrStartNewJob(
      GlideContext glideContext,
      Object model,
      Key signature,
      int width,
      int height,
      Class<?> resourceClass,
      Class<R> transcodeClass,
      Priority priority,
      DiskCacheStrategy diskCacheStrategy,
      Map<Class<?>, Transformation<?>> transformations,
      boolean isTransformationRequired,
      boolean isScaleOnlyOrNoTransform,
      Options options,
      boolean isMemoryCacheable,
      boolean useUnlimitedSourceExecutorPool,
      boolean useAnimationPool,
      boolean onlyRetrieveFromCache,
      ResourceCallback cb,
      Executor callbackExecutor,
      EngineKey key,
      long startTime) {
    // EngineJob是管理加载的任务，先从缓存查找，有的话就复用
    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
    if (current != null) {
      current.addCallback(cb, callbackExecutor);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Added to existing load", startTime, key);
      }
      return new LoadStatus(cb, current);
    }
    // 没有缓存，则创建新任务
    EngineJob<R> engineJob =
        engineJobFactory.build(
            key,
            isMemoryCacheable,
            useUnlimitedSourceExecutorPool,
            useAnimationPool,
            onlyRetrieveFromCache);
    // 创建用于资源解码的任务
    DecodeJob<R> decodeJob =
        decodeJobFactory.build(
            glideContext,
            model,
            key,
            signature,
            width,
            height,
            resourceClass,
            transcodeClass,
            priority,
            diskCacheStrategy,
            transformations,
            isTransformationRequired,
            isScaleOnlyOrNoTransform,
            onlyRetrieveFromCache,
            options,
            engineJob);

    jobs.put(key, engineJob);

    engineJob.addCallback(cb, callbackExecutor);
    // 开始执行
    engineJob.start(decodeJob);

    if (VERBOSE_IS_LOGGABLE) {
      logWithTimeAndKey("Started new load", startTime, key);
    }
    return new LoadStatus(cb, engineJob);
}
```

### 通过EngineJob执行DecodeJob
执行engineJob的start方法：
```
// EngineJob#start

public synchronized void start(DecodeJob<R> decodeJob) {
    this.decodeJob = decodeJob;
    GlideExecutor executor =
        decodeJob.willDecodeFromCache() ? diskCacheExecutor : getActiveSourceExecutor();
    executor.execute(decodeJob);
}
```
diskCacheExecutor来源于构造Glide时初始化Engine传入，getActiveSourceExecutor()也是选择特定线程池，继续看执行DecodeJob的run方法：
```
  public void run() {
    //...
    DataFetcher<?> localFetcher = currentFetcher; //runWrapped()中可能会赋值currentFetcher，所以这里保存一个本地变量
    try {
      //...
      runWrapped();
    } catch (CallbackException e) {
      //...
    } catch (Throwable t) {
      //...
    } finally {
      // Keeping track of the fetcher here and calling cleanup is excessively paranoid, we call
      // close in all cases anyway.
      if (localFetcher != null) {
        localFetcher.cleanup();
      }
      GlideTrace.endSection();
    }
  }
```
核心在于执行runWrapped()：
```
private void runWrapped() {
    switch (runReason) {
      case INITIALIZE:
        stage = getNextStage(Stage.INITIALIZE);
        currentGenerator = getNextGenerator();
        runGenerators();
        break;
      case SWITCH_TO_SOURCE_SERVICE:
        runGenerators();
        break;
      case DECODE_DATA:
        decodeFromRetrievedData();
        break;
      default:
        throw new IllegalStateException("Unrecognized run reason: " + runReason);
    }
}
```
1. getNextStage会根据缓存策略，得到下一步的状态，getNextGenerator()将根据此状态得到对应的DataFetcherGenerator。
2. runGenerators()执行得到的DataFetcherGenerator的startNext()方法。
3. 如果startNext()返回false，表示此DataFetcherGenerator无法加载资源，那么runGenerators()内部会重复第1和第2步，找到另外的DataFetcherGenerator继续执行。

先不考虑缓存，我们直接看SourceGenerator的startNext方法：
```
// SourceGenerator#startNext
public boolean startNext() {
    if (dataToCache != null) {
      Object data = dataToCache;
      dataToCache = null;
      cacheData(data);
    }

    if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
      return true;
    }
    sourceCacheGenerator = null;

    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
      loadData = helper.getLoadData().get(loadDataListIndex++);
      if (loadData != null
          && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
              || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
        started = true;
        startNextLoad(loadData);
      }
    }
    return started;
}
```
### 选取ModelLoader
这里要先了解一下ModelLoader。在Glide初始化的时候，会调用registry来注册ModelLoaderFactory：
```
// Glide构造函数

registry
        ...//省略
        .append(String.class, InputStream.class, new DataUrlLoader.StreamFactory<String>())
        .append(Uri.class, InputStream.class, new DataUrlLoader.StreamFactory<Uri>())
        .append(String.class, InputStream.class, new StringLoader.StreamFactory())
        .append(String.class, ParcelFileDescriptor.class, new StringLoader.FileDescriptorFactory())
        .append(
            String.class, AssetFileDescriptor.class, new StringLoader.AssetFileDescriptorFactory())
        .append(Uri.class, InputStream.class, new HttpUriLoader.Factory())
        ...//省略
```
传入的三个参数，第一个就是我们需要请求的资源数据类型，比如"http://hfy.com/photo/home.jpg"就是字符串类型，第二个是得到的请求结果的类型，比如InputStream.class，第三个是构造ModelLoader的工厂对象，三个参数会构造为MultiModelLoaderFactory.Entry数据结构。

以我们传入字符串为例，可以匹配四种工厂对象：`DataUrlLoader.StreamFactory<String>`、`StringLoader.AssetFileDescriptorFactory`、`StringLoader.StreamFactory`和`StringLoader.FileDescriptorFactory `。分别会构造`DataUrlLoader`、`StringLoader`、`StringLoader`、`StringLoader`，虽然后面三种都是`StringLoader`，但实际它们分别是三种Uri类型的ModelLoader的包装。所以会继续匹配Uri类型，Uri类型的配置如下：
```
.append(Uri.class, InputStream.class, new DataUrlLoader.StreamFactory<Uri>())
.append(Uri.class, InputStream.class, new HttpUriLoader.Factory())
.append(Uri.class, InputStream.class, new AssetUriLoader.StreamFactory(context.getAssets()))
.append(Uri.class, ParcelFileDescriptor.class, new AssetUriLoader.FileDescriptorFactory(context.getAssets()))
.append(Uri.class, InputStream.class, new MediaStoreImageThumbLoader.Factory(context))
.append(Uri.class, InputStream.class, new MediaStoreVideoThumbLoader.Factory(context));
.append(Uri.class, InputStream.class, new UriLoader.StreamFactory(contentResolver))
.append(Uri.class, ParcelFileDescriptor.class, new UriLoader.FileDescriptorFactory(contentResolver))
.append(Uri.class, AssetFileDescriptor.class, new UriLoader.AssetFileDescriptorFactory(contentResolver))
.append(Uri.class, InputStream.class, new UrlUriLoader.StreamFactory())
//省略少数
```
所以传入String，会通过MultiModelLoaderFactory构造出一个MultiModelLoader，这是多个ModelLoader的包装类型。在SourceGenerator中通过DecodeHelper获取到该MultiModelLoader，然后调用它的buildLoadData方法，其中把可以处理该uri的多个ModelLoader对应的DataFetcher组合为MultiFetcher，并构造为LoadData返回。

### 回到SourceGenerator#startNextLoad
在startNext中，得到loadData后会执行startNextLoad：
```
private void startNextLoad(final LoadData<?> toStart) {
    loadData.fetcher.loadData(
        helper.getPriority(),
        new DataCallback<Object>() {
          @Override
          public void onDataReady(@Nullable Object data) {
            if (isCurrentRequest(toStart)) {
              onDataReadyInternal(toStart, data);
            }
          }

          @Override
          public void onLoadFailed(@NonNull Exception e) {
            if (isCurrentRequest(toStart)) {
              onLoadFailedInternal(toStart, e);
            }
          }
        });
}
```

### 使用DataFetcher加载数据
这里就会执行DataFetcher的loadData方法，这里以HttpUrlFetcher为例：
```
public void loadData(
      @NonNull Priority priority, @NonNull DataCallback<? super InputStream> callback) {
    //...
    try {
      InputStream result = loadDataWithRedirects(glideUrl.toURL(), 0, null, glideUrl.getHeaders());
      callback.onDataReady(result);
    } catch (IOException e) {
      //...
      callback.onLoadFailed(e);
    } finally {
      //...
    }
}
```
这里很简单，就是通过http连接得到InputStream，成功后回调onDataReady，这里的callback就是上一步传入的匿名内部类。

### 执行SourceGenerator#onDataReadyInternal
```
void onDataReadyInternal(LoadData<?> loadData, Object data) {
    DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
    if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
      // 如果可以缓存，先赋值
      dataToCache = data;
      // 这个cb是DecodeJob
      cb.reschedule();
    } else {
      cb.onDataFetcherReady(
          loadData.sourceKey,
          data,
          loadData.fetcher,
          loadData.fetcher.getDataSource(),
          originalKey);
    }
}
```
1. 缓存情况：执行DecodeJob的reschedule，则会让DecodeJob再执行run，这次会直接执行runGenerators()，并再次执行SourceGenerator的startNext方法，但这次的dataToCache不为null，会缓存结果，并使用DataCacheGenerator再获取缓存数据，最终回调DecodeJob的onDataFetcherReady方法。
2. 不缓存情况：直接回调DecodeJob的onDataFetcherReady方法

### DecodeJob#onDataFetcherReady
```
public void onDataFetcherReady(
      Key sourceKey, Object data, DataFetcher<?> fetcher, DataSource dataSource, Key attemptedKey) {
    this.currentSourceKey = sourceKey;
    this.currentData = data;
    this.currentFetcher = fetcher;
    this.currentDataSource = dataSource;
    this.currentAttemptingKey = attemptedKey;
    if (Thread.currentThread() != currentThread) {
      runReason = RunReason.DECODE_DATA;
      callback.reschedule(this);
    } else {
      GlideTrace.beginSection("DecodeJob.decodeFromRetrievedData");
      try {
        decodeFromRetrievedData();
      } finally {
        GlideTrace.endSection();
      }
    }
}
```
如果线程不同，就调度执行run方法，最终会执行decodeFromRetrievedData()，线程相同就直接执行decodeFromRetrievedData()：
```
private void decodeFromRetrievedData() {
    //...
    Resource<R> resource = null;
    try {
      resource = decodeFromData(currentFetcher, currentData, currentDataSource);
    } catch (GlideException e) {
      //...
    }
    if (resource != null) {
      // 拿到resource会一层层回调到Target
      notifyEncodeAndRelease(resource, currentDataSource);
    } else {
      runGenerators();
    }
}
```
### 解码和转换流程
decodeFromData()
-> decodeFromFetcher() //这里面会找到Glide中注册的ResourceDecoder和ResourceTranscoder，包装于LoadPath的DecodePath字段中
 -> runLoadPath()
  -> LoadPath#load()
   -> LoadPath#loadWithExceptionList()
    -> DecodePath#decode()

略过中间流程，直接看到DecodePath#decode()：
```
public Resource<Transcode> decode(
      DataRewinder<DataType> rewinder,
      int width,
      int height,
      @NonNull Options options,
      DecodeCallback<ResourceType> callback)
      throws GlideException {
    // 先解码
    Resource<ResourceType> decoded = decodeResource(rewinder, width, height, options);
    // 再转换
    Resource<ResourceType> transformed = callback.onResourceDecoded(decoded);
    // 然后转换类型
    return transcoder.transcode(transformed, options);
}
```
先看解码流程：
```
private Resource<ResourceType> decodeResource(
      DataRewinder<DataType> rewinder, int width, int height, @NonNull Options options)
      throws GlideException {
    List<Throwable> exceptions = Preconditions.checkNotNull(listPool.acquire());
    try {
      // 执行decodeResourceWithList
      return decodeResourceWithList(rewinder, width, height, options, exceptions);
    } finally {
      listPool.release(exceptions);
    }
}

private Resource<ResourceType> decodeResourceWithList(
      DataRewinder<DataType> rewinder,
      int width,
      int height,
      @NonNull Options options,
      List<Throwable> exceptions)
      throws GlideException {
    Resource<ResourceType> result = null;
    //noinspection ForLoopReplaceableByForEach to improve perf
    for (int i = 0, size = decoders.size(); i < size; i++) {
      ResourceDecoder<DataType, ResourceType> decoder = decoders.get(i);
      try {
        DataType data = rewinder.rewindAndGet();
        if (decoder.handles(data, options)) {
          data = rewinder.rewindAndGet();
          result = decoder.decode(data, width, height, options);
        }
      } catch (IOException | RuntimeException | OutOfMemoryError e) {
        //...
      }

      if (result != null) {
        break;
      }
    }
    //...
    return result;
}
```
关键在于寻找可以处理此数据类型的解码器，这里以StreamBitmapDecoder为例，调用它的decode方法，会把`InputStream`转换为`Resource<Bitmap>`，这里就不再继续深入。

继续看转换流程，在DecodePath的decode方法中，执行callback.onResourceDecoded(decoded)，继而执行DecodeJob的onResourceDecoded方法，其中会取出RequestOption中设置的Transformation，执行变换操作。

## 缓存
### 缓存Key
在Engine#load方法中，有创建EngineKey，它由多个数据组成：
```
EngineKey key =
        keyFactory.buildKey(
            model,
            signature,
            width,
            height,
            transformations,
            resourceClass,
            transcodeClass,
            options);
```
这些参数决定了缓存的Key，比如width和height改变了，也会生成不同的Key。

### 内存缓存
#### 内存缓存原理
Glide默认会开启内存缓存，如果需要禁用，则需要调用skipMemoryCache方法：
```
Glide.with(this).load(url).skipMemoryCache(true).into(view);
```

在Glide初始化时，会创建内存缓存，这是一个使用LRU算法的内存缓存：
```
memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
```

Glide的内存缓存分为：
1. 活动资源 (Active Resources) - 现在是否有另一个 View 正在展示这张图片？
2. 内存缓存 (Memory cache) - 该图片是否最近被加载过并仍存在于内存中？

在
```
private EngineResource<?> loadFromMemory(
      EngineKey key, boolean isMemoryCacheable, long startTime) {
    if (!isMemoryCacheable) {
      return null;
    }

    EngineResource<?> active = loadFromActiveResources(key);
    if (active != null) {
      return active;
    }

    EngineResource<?> cached = loadFromCache(key);
    if (cached != null) {
      return cached;
    }

    return null;
}
```
loadFromActiveResources会从ActiveResources中读取缓存，这个缓存内部是一个值为弱引用的HashMap。如果在ActiveResources中读不到，说明当前Key对应的资源没有正在使用，那么就使用loadFromCache从LRU缓存中读取；如果从LRU中读取到资源，那么先把资源从LRU缓存中移除，再放到ActiveResources缓存中，然后返回，目的是使用ActiveResources来缓存正在使用的图片，可以避免这些资源被LruCache算法回收。

#### 写入内存缓存
```
// Engine#构造函数中

engineJobFactory =
    new EngineJobFactory(
        diskCacheExecutor,
        sourceExecutor,
        sourceUnlimitedExecutor,
        animationExecutor,
        /*engineJobListener=*/ this,
        /*resourceListener=*/ this);
```
在Engine构造函数中，会创建engineJobFactory，这里会传入engineJobListener，也就是Engine自己；在请求完成后，EngineJob会回调onEngineJobComplete：
```
// Engine#onEngineJobComplete

public synchronized void onEngineJobComplete(
      EngineJob<?> engineJob, Key key, EngineResource<?> resource) {
    // A null resource indicates that the load failed, usually due to an exception.
    if (resource != null && resource.isMemoryCacheable()) {
      activeResources.activate(key, resource);
    }

    jobs.removeIfCurrent(key, engineJob);
}
```
这里就会把资源缓存到activeResources中，那什么时候会放到LRU缓存中呢？在view从window中移除后会清除ActiveResources缓存，并放入LRU缓存：CustomViewTarget中注册View的onViewDetachedFromWindow，回调时会执行request.clear() --> engine.release() --> EngineResource.release() --> engine.onResourceReleased()：
```
public void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
    activeResources.deactivate(cacheKey);
    if (resource.isMemoryCacheable()) {
      cache.put(cacheKey, resource);
    } else {
      resourceRecycler.recycle(resource, /*forceNextFrame=*/ false);
    }
}
```
这里就会把从activeResources中移除的缓存放到LRU缓存中。

### 磁盘缓存
#### 磁盘缓存策略
Glide的磁盘缓存策略分为五种：
1. DiskCacheStrategy.NONE： 表示不缓存任何内容。
2. DiskCacheStrategy.RESOURCE： 缓存解码和变换后的资源。
3. DiskCacheStrategy.DATA： 缓存未解码的原始数据。
4. DiskCacheStrategy.ALL ： 使用DATA和RESOURCE缓存远程数据，仅使用RESOURCE来缓存本地数据（因为本地数据就是原始数据，不需要DATA缓存）。
5. DiskCacheStrategy.AUTOMATIC ： 默认策略，加载远程数据，仅存储未被修改（变换、裁剪等）的原始数据；加载本地数据，仅存储变换后的数据。

#### 磁盘缓存类型
1. 资源类型（Resource） - 该图片是否之前曾被解码、转换并写入过磁盘缓存？
2. 数据来源 (Data) - 构建这个图片的资源是否之前曾被写入过文件缓存？

#### DATA缓存的读写
* 先看DATA缓存的写入：前面源码介绍SourceGenerator的startNext方法的时候，有看到cacheData方法，这个方法中就会创建DataCacheKey，并把原始数据写入磁盘LRU缓存中。
* 再看DATA缓存的读取：在DecodeJob的runWrapped方法中，如果支持DATA缓存，那么在判断Resource没有缓存后，会执行DataCacheGenerator的startNext方法，其中会创建DataCacheKey，并从磁盘LRU缓存中读取DATA原始数据缓存文件。

#### Resource缓存的读写
* 先看Resource缓存的写入：完成请求和解码后，将执行DecodeJob的notifyEncodeAndRelease方法，其中会调用deferredEncodeManager的encode方法，这里会把Resource资源写入磁盘LRU缓存。
* 再看Resource缓存的读取：在DecodeJob的runWrapped方法中，如果支持Resource缓存，会执行ResourceCacheGenerator的startNext方法，其中会创建ResourceCacheKey，并从磁盘LRU缓存中读取Resource缓存文件。

## 自定义模块
自定义模块功能将更改Glide配置和替换Glide组件等操作独立出来，更方便地定制化，使用方式如下：

首先声明继承AppGlideModule的子类，并且用@GlideModule注解修饰，经过编译时注解，会生成默认名称为GlideApp的类型，然后可以像普通的Glide类一样使用。
```
@GlideModule
public class MyAppGlideModule extends AppGlideModule {
    @Override
    public void applyOptions(@NonNull Context context, @NonNull GlideBuilder builder) {
        builder.setDiskCache(new ExternalPreferredCacheDiskCacheFactory(context, DISK_CACHE_SIZE));
    }

    @Override
    public void registerComponents(@NonNull Context context, @NonNull Glide glide, @NonNull Registry registry) {
        registry.replace(GlideUrl.class, InputStream.class, new OkHttpUrlLoader.Factory());
    }
}
```
上面的示例，一是给设置Glide设置默认配置，例如缓存策略；二是注册或者替换ModelLoader、Encoder等组件，例如替换网络请求为OkHttp。

## Generated API
Generated API功能需要在使用@GlideModule的前提下使用

使用@GlideExtension和@GlideOption生成自定义的API：
```
@GlideExtension
public class MyExtension {
    private MyExtension() {}

    @GlideOption
    public static BaseRequestOptions<?> myOption(BaseRequestOptions<?>  options) {
        return options
                .centerCrop();
    }
}
```
@GlideOption会给GlideRequest类生成myOption()方法，也会给RequestOptions的子类GlideOptions生成相关方法。当前一般用的是前者：
```
GlideApp.with(this).load(url).myOption().into(imageView);
```

@GlideType用于给RequestManager的子类GlideRequests增加方法，添加对新的解码后的资源类型的支持，Glide默认解码为Drawable类型，下面例子使结果解码为GifDrawable类型：
```
@GlideExtension
public class MyExtension {
    private MyExtension() {
    }

    private static final RequestOptions DECODE_TYPE_GIF = GlideOptions.decodeTypeOf(GifDrawable.class).lock();

    @GlideType(GifDrawable.class)
    public static RequestBuilder<GifDrawable> asGIF(RequestBuilder<GifDrawable> requestBuilder) {
        return requestBuilder
                .transition(new DrawableTransitionOptions())
                .apply(DECODE_TYPE_GIF);
    }
}
```

使用方式是用在GlideApp.with(this)之后：
```
GlideApp.with(this).asGIF().load(url).into(imageView);
```