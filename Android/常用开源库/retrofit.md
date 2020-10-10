## 使用方式
### HTTP请求方法注解
|注解|描述|
|---|---|
|GET|七种注解都分别对应HTTP的请求方法，参数都可以设置相对baseUrl的路径，组合为完整的url。但如果url需要动态传参设置，则可以使用Path或Url注解|
|POST||
|PUT||
|HEAD||
|DELETE||
|PATCH||
|OPTIONS||
|HTTP|HTTP注解是一个通用的HTTP方法注解，可以替换以上任何一种注解|

以GET请求为例，结合Path注解可以动态设置url：
```
interface Api {
    @GET("users/{user}/repos")
    Call<List<Repo>> listRepos(@Path("user") String user);
}
```
如果url需要添加键值对，则可以在参数中使用Query注解。

这里的Call是retrofit中的Call，不是okHttp的Call。默认泛型只能是ResponseBody

### 标记注解
* @FormUrlEncoded：对应 Content-Type:application/x-www-form-urlencoded，需要结合 @Field 注解来设置键值对
* @Multipart：对应 Content-Type:mutilpart/form-data，例如上传文件时使用。需要结合 @Part 或者 @PartMap 使用
* @Streaming：从BuiltInConverters中可见，默认会把原本的ResponseBody对应的数据全部读到OkIo的Buffer缓存中，然后返回一个包含了这个Buffer的新ResponseBody，而使用Streaming注解，则会直接返回原本的ResponseBody

### 请求参数注解
|注解|描述|
|---|---|
|Headers|修饰方法，设置HTTP请求的头部数据|
|Header|修饰参数，设置HTTP请求的头部数据，可以使用列表或数组|
|HeaderMap|修饰参数，设置HTTP请求的头部数据，可以使用Map或okhttp3.Headers|
|Field|配合FormUrlEncoded使用，设置单个键值对|
|FieldMap|配合FormUrlEncoded使用，设置多个键值对|
|Part|配合Multipart使用，设置单个Part，可以使用okhttp3.MultipartBody.Part、okhttp3.RequestBody和其他类型；okhttp3.MultipartBody.Part不用设置表单字段信息，其他需要在注解中包含表单字段信息|
|PartMap|配合Multipart使用，支持okhttp3.RequestBody，其他类型会被Converter转化，例如GsonConverter|
|Body|用于POST/PUT请求方法，注解的对象会被Converter转化为ResponseBody|
|Path|修饰参数，可以动态设置部分url|
|Url|修饰参数，可以动态设置整个相对url|
|Query|设置url中的查询参数，一般用于GET请求|
|QueryMap|设置url中的多个查询参数，一般用于GET请求|
|QueryName|设置url中的不带值的查询参数，一般用于GET请求|

### Converter
```
public interface Converter<F, T> {

  // 转换F类型为T类型
  @Nullable
  T convert(F value) throws IOException;

  /** 基于类型和作用，创建 Converter 实例的工厂 */
  abstract class Factory {
    /**
     * 返回用于转换 ResponseBody 为其他类型（type参数）的 Converter , 如果此工厂不能处理则返回null。
     * 例如声明 Call<SimpleResponse>，就会通过 Converter 把 ResponseBody 转换为 SimpleResponse
     */
    public @Nullable Converter<ResponseBody, ?> responseBodyConverter(
        Type type, Annotation[] annotations, Retrofit retrofit) {
      return null;
    }

    /**
     * 返回用于转换其他类型（type参数）为 RequestBody 类型的 Converter，如果此工厂不能处理则返回null。
     * 这里返回的 Converter 将会把 @Body、@Part和@PartMap 注解的类型转换为 RequestBody
     */
    public @Nullable Converter<?, RequestBody> requestBodyConverter(
        Type type,
        Annotation[] parameterAnnotations,
        Annotation[] methodAnnotations,
        Retrofit retrofit) {
      return null;
    }

    /**
     * 返回用于转换 String 类型为 RequestBody 类型的 Converter，如果此工厂不能处理则返回null。
     * 这里返回的 Converter 将会把 @Field、@FieldMap、@Header、@HeaderMap、@Path、@Query和@QueryMap
     * 注解的类型转换为String。一般不会重写此函数，那么retrofit会使用自身默认的ToStringConverter，也就是
     * 直接调用toString()。
     */
    public @Nullable Converter<?, String> stringConverter(
        Type type, Annotation[] annotations, Retrofit retrofit) {
      return null;
    }

    ...
  }
}
```

### CallAdapter
```
// 用于把 Call<R> 转换为 T 类型
public interface CallAdapter<R, T> {
  /**
   * 返回的类型表示此CallAdapter需要Converter把数据类型转换为这里返回的类型，这个type实际会传递给Converter.Factory#responseBodyConverter方法
   */
  Type responseType();

  /**
   * 返回代理call的T类型实例
   */
  T adapt(Call<R> call);

  /**
   * 基于Api接口方法的返回类型创建 CallAdapter 的工厂类
   */
  abstract class Factory {
    /**
     * 如果支持接口方法的返回类型，就返回对应的 CallAdapter，否则返回null
     */
    public abstract @Nullable CallAdapter<?, ?> get(
        Type returnType, Annotation[] annotations, Retrofit retrofit);

    /**
     * 获取泛型参数的上界类型。例如，Map<String, ? extends Runnable> 的index为1，则返回 Runnable
     */
    protected static Type getParameterUpperBound(int index, ParameterizedType type) {
      return Utils.getParameterUpperBound(index, type);
    }

    /**
     * 获取泛型的原始类型。例如，List<? extends Runnable> 将返回 List.class
     */
    protected static Class<?> getRawType(Type type) {
      return Utils.getRawType(type);
    }
  }
}
```

RxJava

Kotlin协程

## 源码分析
一般使用retrofit流程如下：
```
Retrofit retrofit = new Retrofit.Builder()
      .baseUrl("https://api.hfy.com/")
      .addConverterFactory(GsonConverterFactory.create())
      .addCallAdapterFactory(RxJava3CallAdapterFactory.create())
      .build();
ApiService service = retrofit.create(GitHubService.class);
//然后就可以使用ApiService接口中声明的方法来执行网络请求
```

我们先从Retrofit.Builder的build()方法开始分析：
```
public Retrofit build() {
    if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
    }
    // CallFactory实际就是OkHttpClient
    okhttp3.Call.Factory callFactory = this.callFactory;
    if (callFactory == null) {
        callFactory = new OkHttpClient();
    }
    // 回调的执行者，Android会在主线程执行
    Executor callbackExecutor = this.callbackExecutor;
    if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
    }

    // 组合自行添加的和默认的CallAdapter.Factory，默认的可以保障基本功能
    List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
    callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));

    // 组合所有Converter.Factory
    List<Converter.Factory> converterFactories =
        new ArrayList<>(
            1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

    // 先添加内置的Converter.Factory，？？？？？？
    // ensures correct behavior when using converters that consume all types.
    converterFactories.add(new BuiltInConverters());
    converterFactories.addAll(this.converterFactories);
    converterFactories.addAll(platform.defaultConverterFactories());

    return new Retrofit(
        callFactory,
        baseUrl,
        unmodifiableList(converterFactories),
        unmodifiableList(callAdapterFactories),
        callbackExecutor,
        validateEagerly);
}
```

然后看retrofit的create(Class<T>)方法：
```
public <T> T create(final Class<T> service) {
    validateServiceInterface(service); //检查是接口
    return (T)
        Proxy.newProxyInstance(
            service.getClassLoader(),
            new Class<?>[] {service},
            new InvocationHandler() {
              private final Platform platform = Platform.get();
              private final Object[] emptyArgs = new Object[0];

              @Override
              public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                  throws Throwable {
                // 如果方法声明在Object类中，例如wait()方法，则直接调用
                if (method.getDeclaringClass() == Object.class) {
                  return method.invoke(this, args);
                }
                args = args != null ? args : emptyArgs;
                // 一般不使用接口默认方法，所以执行loadServiceMethod
                return platform.isDefaultMethod(method)
                    ? platform.invokeDefaultMethod(method, service, proxy, args)
                    : loadServiceMethod(method).invoke(args);
              }
            });
}
```

loadServiceMethod方法就是获取一个ServiceMethod实例，然后调用invoke来执行
```
ServiceMethod<?> loadServiceMethod(Method method) {
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;
    // 有缓存就取缓存，没有缓存就创建
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = ServiceMethod.parseAnnotations(this, method);
        serviceMethodCache.put(method, result);
      }
    }
    return result;
}
```
ServiceMethod的静态方法parseAnnotations将创建ServiceMethod实例：
```
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    // 根据方法和参数上的注解，得到用于创建Request的工厂实例，其中也会根据参数类型是否是Continuation
    // 来判定是否是kotlin的挂起函数 
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

    // ...省略对不支持的method的返回类型的处理
    // 实际通过HttpServiceMethod的静态方法parseAnnotations来获取ServiceMethod
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }
```
可见实际是返回ServiceMethod的子类HttpServiceMethod：
```
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
    boolean isKotlinSuspendFunction = requestFactory.isKotlinSuspendFunction;
    boolean continuationWantsResponse = false;
    boolean continuationBodyNullable = false;

    Annotation[] annotations = method.getAnnotations();
    Type adapterType;

    // 如果是Kotlin的挂起函数
    if (isKotlinSuspendFunction) {
      Type[] parameterTypes = method.getGenericParameterTypes();
      // 根据挂起函数中生成的Continuation<T>的泛型参数，得到实际返回类型T
      Type responseType =
          Utils.getParameterLowerBound(
              0, (ParameterizedType) parameterTypes[parameterTypes.length - 1]);
      //如果挂起函数返回的是Response类型
      if (getRawType(responseType) == Response.class && responseType instanceof ParameterizedType) {
        // 就让responseType赋值为 Response<T> 的泛型参数T
        responseType = Utils.getParameterUpperBound(0, (ParameterizedType) responseType);
        // 并且标记挂起函数的返回类型需要是Response，也就是实际数据包装为Response<RealType>
        continuationWantsResponse = true;
      } else {
        ...
      }
      //到这里，responseType就是需要转换的实际类型
      // 赋值adapterType为相关参数化类型
      adapterType = new Utils.ParameterizedTypeImpl(null, Call.class, responseType);
      annotations = SkipCallbackExecutorImpl.ensurePresent(annotations);
    } else {
      // 如果是普通函数，那么直接赋值adapterType为函数返回的类型
      adapterType = method.getGenericReturnType();
    }

    // 根据adapterType等信息，查找得到适合的CallAdapter。
    CallAdapter<ResponseT, ReturnT> callAdapter =
        createCallAdapter(retrofit, method, adapterType, annotations);
    Type responseType = callAdapter.responseType();
    // ...省略对responseType的类型检查
    // 根据相关信息，查找得到适合的Converter。responseType就是CallAdapter需要的Converter帮助转换到的类型
    Converter<ResponseBody, ResponseT> responseConverter =
        createResponseConverter(retrofit, method, responseType);

    okhttp3.Call.Factory callFactory = retrofit.callFactory; // 实际就是OkHttpClient
    if (!isKotlinSuspendFunction) {
      // 普通函数，则得到HttpServiceMethod的子类CallAdapted
      return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
    } else if (continuationWantsResponse) {
      // kotlin挂起函数对应的HttpServiceMethod实例，需要返回Response类型
      return (HttpServiceMethod<ResponseT, ReturnT>)
          new SuspendForResponse<>(
              requestFactory,
              callFactory,
              responseConverter,
              (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter);
    } else {
      // kotlin挂起函数对应的HttpServiceMethod实例
      return (HttpServiceMethod<ResponseT, ReturnT>)
          new SuspendForBody<>(
              requestFactory,
              callFactory,
              responseConverter,
              (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter,
              continuationBodyNullable);
    }
}
```

以普通函数对应的CallAdapted为例，它是HttpServiceMethod的子类，而HttpServiceMethod是ServiceMethod的子类；在开始的动态代理代码中看到，会调用ServiceMethod的invoke方法，实际就是HttpServiceMethod的invoke方法实现：
```
@Override
final @Nullable ReturnT invoke(Object[] args) {
    // 创建OkHttpCall，传入responseConverter等对象。responseConverter将会在实际的请求完成后转换ResponseBody为指定的类型
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
    return adapt(call, args);
}
```
实际调用CallAdapted的adapt方法，这里就会实际调用callAdapter来转换Call为其他类型：
```
@Override
protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
    return callAdapter.adapt(call);
}
```

**流程总结**：通过动态代理，调用接口的方法，通过ServiceMethod的子类来代理。根据注解等信息得到requestFactory，根据相关类型查找callAdapter和converter。将requestFactory、responseConverter等对象传递给OkHttpCall，OkHttpCall（retrofit2.Call的子类）就是实际的OkHttp.Call的包装。得到retrofit2.Call后，交给callAdpater转换类型，最终由开发者触发请求。实际请求结果ResponseBody会被Converter转换为需要的类型。