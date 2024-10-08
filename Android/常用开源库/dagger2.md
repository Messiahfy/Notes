https://developer.android.google.cn/training/dependency-injection/dagger-android#kotlin
https://codelabs.developers.google.com/codelabs/android-dagger
https://dagger.dev/dev-guide/

## 概述
Dagger2是一个依赖注入框架，它会根据开发者使用相关注解写的依赖关系的代码，在编译时生成相关代码，然后开发者可以使用生成的代码完成依赖注入。Dagger2中使用的注解，一部分是属于javax.inject的，例如@Inject；另一部分则是属于Dagger2自行创造的，例如@Component。属于javax.inject的那一部分注解，使用方式须按照Dagger2的要求，所以可能会和javax.inject文档中的使用说明有差异。

基于2.48.1版本分析

## 1. @Inject和@Component
### 使用构造函数注入
```
public class Car {
    private Engine engine;

    @Inject
    public Car(Engine engine) {
        this.engine = engine;
    }
}
```
上面的代码，会告知Dagger2:
1. 使用带有 @Inject 注释的构造函数创建 Car 实例
2. Car 实例依赖 Engine

```
class Engine {
    @Inject
    public Engine() {
    }

    public void start(){
        Log.d("引擎","启动");
    }
}
```
同理，在Engine类里面，告知Dagger2:
1. 使用带有 @Inject 注释的构造函数创建 Engine 实例
2. Engine 实例不依赖其他实例

以上代码描述清楚了构造一个 Car 的依赖关系，那么现在如何使用Dagger2得到一个 Car 实例呢？现在需要用到 @Component 了。
```
@Component
public interface CarComponent {
    Car createCar();
}
```
上面代码用 @Component 注解一个接口，接口命名可以自行决定，接口中是一个返回 Car 实例的方法。因为 Car 类中已经用 @Inject 告知Dagger2如何构造 Car 实例，所以 `createCar` 方法会通过该构造函数来构造 Car，而构造函数需要 Engine，并且 Engine 中也用 @Inject 告知Dagger2构造 Engine 的方式，所以Dagger2可以生成 createCar() 函数的内容：
```
public final class DaggerCarComponent {
  private DaggerCarComponent() {
  }

  public static Builder builder() {
    return new Builder();
  }

  public static CarComponent create() {
    return new Builder().build();
  }

  public static final class Builder {
    private Builder() {
    }

    public CarComponent build() {
      return new CarComponentImpl();
    }
  }

  private static final class CarComponentImpl implements CarComponent {
    private final CarComponentImpl carComponentImpl = this;

    private CarComponentImpl() {

    }

    @Override
    public Car createCar() {
      return new Car(new Engine());
    }
  }
}
```
这是 @Component 注解的接口生成的类，它实现了 CarComponent 接口的`createCar()`函数，也就是通过我们用 `@Inject` 注解的构造函数来构造对应的实例。

然后我们就可以使用如下代码来得到 Car 实例：
```
Car car = DaggerCarComponent.create().createCar();

//或者如下方式，使用@Module的情况下，可以自行构造Module传入Builder，达到给Module传参或者其他目的。。。。
Car car = DaggerCarComponent.builder().build().createCar();
```

综上，我们可以知道 @Component 的作用就是作为我们得到实例的媒介。

> @Inject最多只能注解一个构造函数    

### 使用字段赋值注入
用 `@Inject` 修饰字段的作用，是在运行时向实例注入其内部用`@Inject` 修饰的字段，而不是创建实例本身。

例如Android的Activity实例由系统创建，我们不能通过Dagger2来创建它本身，而是通过 `@Inject` 修饰Activity内部的字段，在运行时注入这些字段的实例对象。

现在修改一下 CarComponent：
```
@Component
public interface CarComponent {
    Car createCar(); 

    void inject(MainActivity activity);
}
```
inject方法可以任意命名，由于它的作用是注入依赖，所以一般叫做inject。参数为MainActivity类型，表示此方法用于向MainActivity的实例注入其内部被 `@Inject` 修饰的字段。

> createCar方法是可选的。如果保留，那么给activity的car赋值时就使用生成的createCar方法；如果没有保留，那么不会生成createCar方法，赋值时就使用生成的getCar方法，其内容和createCar方法一致。

```
public class MainActivity extends AppCompatActivity {
    @Inject
    Car car1;
    @Inject
    Car car2;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        DaggerCarComponent.create().inject(this);
        car1.start();
        car2.start();
    }
}
```
MainActivity类中用 `@Inject` 修饰了两个 Car 字段，然后在使用生成的代码后，car1 和 car2 字段就被赋值了，然后就可以开始使用。

通过生成的代码，可以看到 inject 方法就是给 MainActivity 实例中的 car 字段赋值，生成的代码如下：
```
public final class DaggerCarComponent {

  //......

  private static final class CarComponentImpl implements CarComponent {
    private final CarComponentImpl carComponentImpl = this;

    private CarComponentImpl() {

    }

    @Override
    public Car createCar() {
      return new Car(new Engine());
    }

    // 调用此方法即可注入依赖
    @Override
    public void inject(MainActivity activity) {
      injectMainActivity(activity);
    }

    // 注入各个字段
    private MainActivity injectMainActivity(MainActivity instance) {
      MainActivity_MembersInjector.injectCar1(instance, createCar());
      MainActivity_MembersInjector.injectCar2(instance, createCar());
      return instance;
    }
  }
}

// 实际的注入类
public final class MainActivity_MembersInjector implements MembersInjector<MainActivity> {
  private final Provider<Car> car1Provider;

  private final Provider<Car> car2Provider;

  public MainActivity_MembersInjector(Provider<Car> car1Provider, Provider<Car> car2Provider) {
    this.car1Provider = car1Provider;
    this.car2Provider = car2Provider;
  }

  public static MembersInjector<MainActivity> create(Provider<Car> car1Provider,
      Provider<Car> car2Provider) {
    return new MainActivity_MembersInjector(car1Provider, car2Provider);
  }

  @Override
  public void injectMembers(MainActivity instance) {
    injectCar1(instance, car1Provider.get());
    injectCar2(instance, car2Provider.get());
  }

  // 这里的例子只用到下面两个方法
  @InjectedFieldSignature("com.example.myapplication.MainActivity.car1")
  public static void injectCar1(MainActivity instance, Car car1) {
    instance.car1 = car1;
  }

  @InjectedFieldSignature("com.example.myapplication.MainActivity.car2")
  public static void injectCar2(MainActivity instance, Car car2) {
    instance.car2 = car2;
  }
}

```
注入的方式，会生成 `MembersInjector` 用于注入依赖

> 字段不能使用 `private` 修饰，因为dagger2会生成同一个包内的代码，`非private` 修饰的字段才可以被访问到。

### 使用方法注入
相对于构造函数注入和字段注入，方法注入使用较少。
```
public class Car {
    private Engine engine;
    private Engine2 engine2;

    @Inject
    public Car(Engine engine) {
        this.engine = engine;
    }

    @Inject
    public void setEngine2(Engine2 engine2) {
        this.engine2 = engine2;
    }

    public void start() {
        engine.start();
        engine2.start();
    }
}
```
用 @Inject 修饰setEngine2方法，告知 Dagger2 构造 Car 时，要调用setEngine2方法，传入Engine2实例，Engine2的实例也由Dagger2构造。方法注入和其他注入方式可以一起使用，当然，需要各自注入各自的实例，如果注入相同实例，那么就会注入两次，后面注入的会覆盖前面注入的实例。

-----------------------

总结：@Inject修饰构造函数，告知Dagger2构造此实例的方式，并且构造函数的参数就是依赖项；修饰字段，告知Dagger2依赖项；修饰setter方法，告知Dagger2依赖项。

## 2. @Module和@Provides
从@Inject的使用方式来看，通过Dagger2构造实例，就要用@Inject来修饰该类的构造函数，但是我们用到的第三方库，我们无法修改代码，也就无法用 @Inject 来修饰构造函数。这种情况下，要让Dagger2知道怎么构造我们需要的实例，就要用到@Module和@Provides。

以获取一个OkHttpClient为例：
```
@Module
class HttpModule {
    @Provides
    OkHttpClient provideOkHttpClient() {
        return new OkHttpClient.Builder().build();
    }
}
```
@Module修饰类，@Provides修饰方法，表示OkHttpClient的实例可以通过这里的provideOkHttpClient方法得到，实质就是提供了无法使用@Inject修饰构造函数的替代方式。然后需要确定此Module用于哪个Component：
```
@Component(modules = {HttpModule.class})
public interface HttpComponent {
    OkHttpClient createHttpClient();
}
```
这里我们在HttpComponent中声明返回OkHttpClient的方法，并且在@Component注解中加上HttpModule，现在就可以得到OkHttpClient实例了。当然同样可以使用inject风格的方法，并在Activity中用@Inject修饰字段的方式来注入。

生成代码如下：
```
public final class DaggerCarComponent {
  // ......

  // 和没有使用Module相比，这里的Builder多了 MyModule 字段
  public static final class Builder {
    private MyModule myModule;

    private Builder() {
    }

    // 注意这里，Module都是可以开发者自己去传入实例
    public Builder myModule(MyModule myModule) {
      this.myModule = Preconditions.checkNotNull(myModule);
      return this;
    }

    public CarComponent build() {
      if (myModule == null) {
        // 如果没有自己传入 MyModule ，这里就会创建默认的
        this.myModule = new MyModule();
      }
      return new CarComponentImpl(myModule);
    }
  }

  private static final class CarComponentImpl implements CarComponent {
    private final MyModule myModule;

    private final CarComponentImpl carComponentImpl = this;

    private CarComponentImpl(MyModule myModuleParam) {
      this.myModule = myModuleParam;

    }

    private Car car() {
      return new Car(new Engine());
    }

    @Override
    public void inject(MainActivity activity) {
      injectMainActivity(activity);
    }

    private MainActivity injectMainActivity(MainActivity instance) {
      MainActivity_MembersInjector.injectCar(instance, car());
      // 注入OkHttpClient，实例从 MyModule_ProvideOkHttpClientFactory 中获取
      MainActivity_MembersInjector.injectOkHttpClient(instance, MyModule_ProvideOkHttpClientFactory.provideOkHttpClient(myModule));
      return instance;
    }
  }
}

// 实质就是通过 MyModule 获取实例
public final class MyModule_ProvideOkHttpClientFactory implements Factory<OkHttpClient> {
  private final MyModule module;

  public MyModule_ProvideOkHttpClientFactory(MyModule module) {
    this.module = module;
  }

  @Override
  public OkHttpClient get() {
    return provideOkHttpClient(module);
  }

  public static MyModule_ProvideOkHttpClientFactory create(MyModule module) {
    return new MyModule_ProvideOkHttpClientFactory(module);
  }

  public static OkHttpClient provideOkHttpClient(MyModule instance) {
    return Preconditions.checkNotNullFromProvides(instance.provideOkHttpClient());
  }
}
```

> 如果同时存在 @Module+@Provides的方式提供实例 和 @Inject构造函数，Dagger2内部会优先选择@Module+@Provides提供的实例。

> **Component生成的代码，Module对象可以自行创建传入，否则使用默认构造的**

Module其实就相当于一个获取实例的工厂

**@Provides修饰的方法也可以包含参数，Dagger2会寻找本Module内返回为该参数类型且被@Provides修饰的方法，如果没有，就寻找该类是否有被@Inject修饰的构造函数**

> @Module注解可以使用 includes 属性，作用就是复用Module，相当于把包含的一个或多个Module中提供的实例获取方式都组合到了当前Module。这个包含动作是递归的，包含的Module又包含了其他Module，则都会组合到当前Module。<br/>

使用了 @Module(includes = [AnotherModule::class])，生成代码如下：
```
public final class DaggerCarComponent {
  //......

  // 实际就是 Module 和它 include 的 Module 都被 Component 持有
  public static final class Builder {
    private MyModule myModule;

    private AnotherModule anotherModule;

    private Builder() {
    }

    public Builder myModule(MyModule myModule) {
      this.myModule = Preconditions.checkNotNull(myModule);
      return this;
    }

    public Builder anotherModule(AnotherModule anotherModule) {
      this.anotherModule = Preconditions.checkNotNull(anotherModule);
      return this;
    }

    public CarComponent build() {
      if (myModule == null) {
        this.myModule = new MyModule();
      }
      if (anotherModule == null) {
        this.anotherModule = new AnotherModule();
      }
      return new CarComponentImpl(myModule, anotherModule);
    }
  }

  private static final class CarComponentImpl implements CarComponent {
    private final MyModule myModule;

    private final AnotherModule anotherModule;

    private final CarComponentImpl carComponentImpl = this;

    private CarComponentImpl(MyModule myModuleParam, AnotherModule anotherModuleParam) {
      this.myModule = myModuleParam;
      this.anotherModule = anotherModuleParam;

    }

    private Car car() {
      return new Car(new Engine());
    }

    @Override
    public void inject(MainActivity activity) {
      injectMainActivity(activity);
    }

    private MainActivity injectMainActivity(MainActivity instance) {
      MainActivity_MembersInjector.injectCar(instance, car());
      // 分别通过两个Module获取实例
      MainActivity_MembersInjector.injectOkHttpClient(instance, MyModule_ProvideOkHttpClientFactory.provideOkHttpClient(myModule));
      MainActivity_MembersInjector.injectInt(instance, AnotherModule_ProvideIntFactory.provideInt(anotherModule));
      return instance;
    }
  }
}
```

**使用 @Module 注解的 includes 属性，效果和 @Component 的 modules 数组包含多个 Module 的效果一样**

-----------------------

前面对实例的依赖关系都是用具体的类型来决定的，但是我们开发中很多时候都是面向接口编程，实例的类型都是接口，这种情况下，我们可以也通过 @Provides 来完成依赖注入：
```
class Car @Inject constructor(engine: IEngine) {
    private val engine: IEngine

    init {
        this.engine = engine
    }

    fun start() {
        engine.start()
    }
}

interface IEngine {
    fun start()
}

class Engine @Inject constructor() : IEngine {
    override fun start() {
        Log.d("引擎", "启动")
    }
}
```
现在engine的类型是接口IEngine，要得到实例，无法通过@Inject，我们现在使用@Provides来得到IEngine实例：
```
@Module
class CarModule {

    @Provides
    IEngine provideEngine() {
        return new Engine();
    }

    //上面或者下面的方法可以视情况选择。下面这种适合Engine也需要Dagger2帮我们注入依赖项的情况
    @Provides
    IEngine provideEngine(Engine engine) {
        return engine;
    }
}
```

生成代码如下
```
public final class DaggerCarComponent {
  //......

  private static final class CarComponentImpl implements CarComponent {
    private final MyModule myModule;

    private final CarComponentImpl carComponentImpl = this;

    private CarComponentImpl(MyModule myModuleParam) {
      this.myModule = myModuleParam;

    }

    private IEngine iEngine() {
      // 因为 Engine 的构造函数使用了 Inject 注解，所以通过构造函数创建实例
      return MyModule_ProvideEngineFactory.provideEngine(myModule, new Engine());
    }

    private Car car() {
      // 生成Car时，调用 iEngine() 函数
      return new Car(iEngine());
    }

    @Override
    public void inject(MainActivity activity) {
      injectMainActivity(activity);
    }

    private MainActivity injectMainActivity(MainActivity instance) {
      MainActivity_MembersInjector.injectCar(instance, car());
      return instance;
    }
  }
}


public final class MyModule_ProvideEngineFactory implements Factory<IEngine> {
  //......

  public static IEngine provideEngine(MyModule instance, Engine engine) {
    // 将 engine 传给了 CarModule 的 provideEngine 方法，provideEngine 中我们是直接返回，所以拿到的也就是这个engine
    return Preconditions.checkNotNullFromProvides(instance.provideEngine(engine));
  }
}
```


## 3. @Binds
@Binds和@Provides的作用类似，但是要求Module是抽象类或者接口，修饰的方法是抽象方法，并且需要有类型为返回类型的子类的传入参数。返回类型对应需要注入的位置声明的类型，传入参数的类型为实际要注入的类型。比@Provides可以简化一步 return 过程。
```
@Module
abstract class CarModule {

    @Binds
    abstract IEngine getEngine(Engine engine);
}
```

生成代码如下：
```
public final class DaggerCarComponent {
  // ......

  private static final class CarComponentImpl implements CarComponent {
    private final CarComponentImpl carComponentImpl = this;

    private CarComponentImpl() {


    }

    // 相对于使用 @Provides，因为@Binds要求在抽象方法或接口中使用，所以没有方法实现，
    // 也就不用传入Module的方法中，而是直接使用构造的实例即可
    private Car car() {
      return new Car(new Engine());
    }

    @Override
    public void inject(MainActivity activity) {
      injectMainActivity(activity);
    }

    private MainActivity injectMainActivity(MainActivity instance) {
      MainActivity_MembersInjector.injectCar(instance, car());
      return instance;
    }
  }
}
```


使用 @Binds，可以在构造实例不方便手动构造时使用，比如这个实例也有很多依赖项，这个实例也由Dagger构造的情况

## 4. @Qualifier 和 @Named
前面的依赖注入方式，都是以类型来确定注入依赖的构造方式。但是在一些情况下，构造一个类型的实例的方式有多种，比如两个@Provides修饰的方法返回相同类型的不同实例（比如传入构造参数不同，或者比如一个实例为全局单例、另一个为Activity范围的实例），此时Dagger2就无法知道选择哪个方法来得到实例。这种情况下，我们可以通过 @Qualifier 或者 @Name 来确定选择哪个。

先来自定义一个 @Qualifier：
```
@Qualifier
@Documented
@Retention(RUNTIME)
public @interface Benz {
    int value() default 0;
}
```
这个@Benz注解，带一个int属性（根据自身需求，可以有任意属性、任意类型）。相同的注解，不同的属性就可以区分开使用哪个实例。我们现在Module中对提供不同实例的方法加上@Benz，并且属性值分别为1和2：
```
@Module
class CarModule {

    @Provides
    @Benz(1)
    Car provideCar1() {
        return new Car(new Engine("奔驰"));
    }

    @Provides
    @Benz(2)
    Car provideCar2() {
        return new Car(new Engine("特斯拉"));
    }
}
```
如果我们直接通过Component获取实例，那么就在方法中加上注解，如下：
```
@Component(modules = {CarModule.class})
public interface CarComponent {
    @Benz(1)
    Car createCar1();

    @Benz(2)
    Car createCar2();
}
```
如果我们是通过Component注入到已有对象的字段，那么使用方式如下：
```
@Component(modules = {CarModule.class})
public interface CarComponent {

    void inject(MainActivity activity);
}


public class MainActivity extends AppCompatActivity {
    @Inject
    @Benz(1)
    Car car1;
    @Inject
    @Benz(2)
    Car car2;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        DaggerCarComponent.create().inject(this);
        car1.start();
        car2.start();
    }
}
```

-----------------------

除了用同一个注解的不同属性来区分选择不同的实例，还可以定义多个不同的注解，直接用不同的注解来区分。
```
@Qualifier
@Documented
@Retention(RUNTIME)
public @interface Benz {
}

@Qualifier
@Documented
@Retention(RUNTIME)
public @interface Tesla {
}
```
用法类似：
```
@Module
class CarModule {

    @Provides
    @Benz
    Car provideCar1() {
        return new Car(new Engine("奔驰"));
    }

    @Provides
    @Tesla
    Car provideCar2() {
        return new Car(new Engine("特斯拉"));
    }
}
```
然后后续步骤同样是在需要注入的位置加上注解。

而@Named 就是javax.inject中自带的自定义的 @Qualifier，用法一致。

> 生成代码时，就是根据注解，选择指定的对象提供方式，比较简单，就不写出生成的代码了

## 5. Lazy 和 Provider
Lazy是Dagger2中的一个接口，可以实现懒加载。用它来包装我们需要的实例类型，即可达到注入时不会执行实例化，而要在我们调用Lazy的get方法时才会执行实例化：
```
public class MainActivity extends AppCompatActivity {
    @Inject
    Lazy<Car> lazyCar;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        DaggerCarComponent.create().inject(this);
        lazyCar.get().start();
    }
}
```

Provider是javax.inject中的接口，如果需要多次获得获取实例，则用它包装Car类型，使用时调用get方法就会执行我们用@Inject或者@Provides修饰的获取实例方式。
```
public class MainActivity extends AppCompatActivity {
    @Inject
    Provider<Car> carProvider;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        DaggerCarComponent.create().inject(this);
        Car car1 = carProvider.get();
        Car car2 = carProvider.get();
        Car car3 = carProvider.get();
    }
}
```

生成的代码，会创建一个Provder，子类是Factory
```
public final class DaggerCarComponent {
  //......

  private static final class CarComponentImpl implements CarComponent {
    private final CarComponentImpl carComponentImpl = this;

    private Provider<Car> carProvider;

    private CarComponentImpl() {

      initialize();

    }

    @SuppressWarnings("unchecked")
    private void initialize() {
      // 创建Provider
      this.carProvider = Car_Factory.create(((Provider) Engine_Factory.create()));
    }

    @Override
    public void inject(MainActivity activity) {
      injectMainActivity(activity);
    }

    private MainActivity injectMainActivity(MainActivity instance) {
      // 把Provider注入到MainActivity的carProvider字段
      MainActivity_MembersInjector.injectCar(instance, carProvider);
      return instance;
    }
  }
}
```

> 直接使用@Inject构造函数注入，会生成直接调用构造函数的代码创建对象，而如果使用Provider，则会生成对应的Factory类。**而前面使用@Module配合@Provides的情况即使没有使用Provider也会生成Factory类，但在没有使用Provider时就直接通过生成的Factory构造实例，使用Provider的话就注入此Factory类给Provider包装对象，相当于生成一套Factory类代码两种情况都可以用上**

## 6. 作用域
默认情况下，Component 每次调用 Module 中的 provide 方法或 Inject 构造函数时都会创建一个新的实例，如果我们需要单例，就要用到作用域注解。作用域机制可以保证在`@Scope`标记的`Component`作用域内，类为单例。

@Singleton是javax.inject自带的作用域注解，这个名称没有实际意义，我们也可以用@Scope自定义任何名称的作用域注解。@Singleton的源码如下：
```
@Scope
@Documented
@Retention(RUNTIME)
public @interface Singleton {}
```

作用域注解只能标注目标类（被@Inject标注构造函数的类）、@Provides（@Binds） 方法和 Component。也就是说，作用域注解要生效的话，需要同时标注在 Component 和依赖注入实例的提供者。

我们先在Car类上加上@Singleton：
```
@Singleton
public class Car {
    private Engine engine;

    @Inject
    public Car(Engine engine) {
        this.engine = engine;
    }

    public void start() {
        engine.start();
    }
}
```

然后在Component上加上@Singleton：
```
@Singleton
@Component()
public interface CarComponent {
    void inject(MainActivity activity);
}
```
然后，如果MainActivity中有个Car字段需要注入，那么注入的会是同一个实例，而不会每次都构造新的实例。

```
public final class DaggerCarComponent {
  // ......

  private static final class CarComponentImpl implements CarComponent {
    private final CarComponentImpl carComponentImpl = this;

    private Provider<Car> carProvider;

    private CarComponentImpl(AnotherModule anotherModuleParam) {

      initialize(anotherModuleParam);

    }

    @SuppressWarnings("unchecked")
    private void initialize(final AnotherModule anotherModuleParam) {
      // 使用作用域注解产生的作用就是，生成的provider会被包装在 DoubleCheck 中，实现Component实例范围内的单例
      this.carProvider = DoubleCheck.provider(Car_Factory.create(((Provider) Engine_Factory.create())));
    }

    @Override
    public void inject(MainActivity activity) {
      injectMainActivity(activity);
    }

    private MainActivity injectMainActivity(MainActivity instance) {
      MainActivity_MembersInjector.injectCar(instance, carProvider.get());
      return instance;
    }
  }
}
```
使用作用域注解，将会使用 DoubleCheck 包装生成对象的 factory，实现Component范围内的单例

* Component 可以有多个作用域注解，也就是说 Component 可以和多个不同的作用域的依赖注入实例的提供者绑定使用。
* 依赖注入实例的提供者的作用域注解必须和绑定的 Component 的其中一个作用域注解一样。
* Component 的作用域注解不要求必须存在对应一样注解的依赖注入实例的提供者，反之则如上一条所述。比如 Component 使用了几个作用域注解，但和这个Component相关的依赖注入提供者可以没有作用域注解，此时这些提供者对应生成的代码都不会使用 DoubleCheck 保障单例。


？？？？？？
3. component的dependencies与component自身的scope不能相同：
4. Singleton的组件不能依赖其他的scope的组件，只能其他scope的组件依赖Singleton的组件。因为一般Singleton用于全局单例，如果它依赖其他scope的组件，会导致其他scope的组件生命周期也为全局。
5. 没有scope的component不能依赖有scope的component

作用域注解后的单例都是局部单例，仅在同一个Component实例范围内是单例，在生成的DaggerXxxComponent中，会用同一个DoubleCheck来获取对象。要实现全局单例，就要用同一个Component，比如在Application中缓存一个Component，然后都用这个Component。

## 7. @Reusable 
@Reusable也是一种作用域注解，用于标记依赖注入的提供者，它提供的复用，不是绝对的同步单例，比如创建实例比较是比较重量级的操作，不需要严格单例，但也不想创建太多新实例。而是一个“尽量复用”逻辑。和@Singleton或者其他自定义作用域注解不同，@Reusable **不需要**Component中也使用此注解。

> 生成的代码中，会使用SingleCheck来包装factory，实际就是线程不安全的单例

## 8. 可选绑定 @BindsOptionalOf
```
@BindsOptionalOf abstract Car optionalCar();
```
在Module中使用 @BindsOptionalOf，在需要注入Car的地方，类型可以使用：
* Optional<Car>
* Optional<Provider<Car>>
* Optional<Lazy<Car>>
* Optional<Provider<Lazy<Car>>>
如果，组件中可以找到 Car 实例的提供者，如@Inject修饰的Car的构造函数，或者Module中的@Provides等方式，那么可以通过Optional得到 Car；如果找不到，那么通过 Optional 得到的Car就是null。


## 9. @BindsInstance、@Component.Builder、@Component.Factory
前面的源码可以看到，Component生成的源码中有Builder相关代码，可以设置Module对象。我们可以通过`@Component.Builder`或`@Component.Factory`自定义生成的Builder或者Factory方法，以便自定义我们想传入的参数。

先看看Builder类型的规则要求：
* 必须有一个声明返回指定Component类型或它的父类的抽象无参方法，实际就是对应的build方法
* 可以有其他抽象方法，也就是setter方法
* setter方法必须有一个参数并返回void、Builder类型或者Builder的父类型。如果返回void就不能连续链式调用了，所以一般返回Builder类型更方便使用。
* 如果Component注解设置了dependencies，每个dependency都要有对应的setter方法
* 如果Component注解设置了modules，对于每个Module，如果该Module没有可见的无参构造方法导致Component无法构造该Module，就需要给这个Module添加setter方法。如果是例如使用`@Binds`的Module抽象类或接口，不能配置setter方法。
* 可以有使用`@BindsInstance`注解的setter方法，这个方法只能有一个参数，外部传入的这个参数将被用于注入到Component中需要它的地方。效果和使用Module有参构造函数传入实例并使用Provides来提供它一样。
* 非抽象方法将被忽略


所以如果Component中的依赖项或者其他参数需要从外部传入，我们既可以用Component的Builder传入自行构造Module的方式；同样的目的，我们也可以使用@BindsInstance。

现在Car依赖的Engine的构造函数需要传入字符串参数：
```
public class Engine {
    private String name;

    @Inject
    public Engine(String name) {
        this.name = name;
    }

    public void start() {
        Log.d("引擎", name + " 启动");
    }
}
```
如果我们希望这个name由外部传入，我们就可以使用在Component中使用@BindsInstance：
```
@Component()
public interface CarComponent {
    void inject(MainActivity activity);

    @Component.Builder
    interface Builder {
        @BindsInstance
        Builder engineName(String engineName);

        CarComponent build();
    }
}
```
现在生成的Component代码中不会再有create()方法，Dagger会生成一个CarComponent.Builder的Builder子类，必须使用Builder来构造CarComponent，此时Builder会有一个engineName(String)方法用于传入该参数:
```
DaggerCarComponent.builder().engineName("传入的引擎名称").build().inject(this);
```
生成的源码如下：
```
public final class DaggerCarComponent {
  private DaggerCarComponent() {
  }

  public static CarComponent.Builder builder() {
    return new Builder();
  }

  // 生成的Builder
  private static final class Builder implements CarComponent.Builder {
    private String engineName;

    // 生成了我们使用 @BindsInstance 声明的engineName方法
    @Override
    public Builder engineName(String engineName) {
      this.engineName = Preconditions.checkNotNull(engineName);
      return this;
    }

    @Override
    public CarComponent build() {
      Preconditions.checkBuilderRequirement(engineName, String.class);
      return new CarComponentImpl(engineName);
    }
  }

  private static final class CarComponentImpl implements CarComponent {
    private final CarComponentImpl carComponentImpl = this;

    private Provider<String> engineNameProvider;

    private Provider<Engine> engineProvider;

    private Provider<Car> carProvider;

    private CarComponentImpl(String engineNameParam) {

      initialize(engineNameParam);

    }

    // 传入的engineName用于这里构造实例Engine
    @SuppressWarnings("unchecked")
    private void initialize(final String engineNameParam) {
      this.engineNameProvider = InstanceFactory.create(engineNameParam);
      this.engineProvider = Engine_Factory.create(engineNameProvider);
      this.carProvider = SingleCheck.provider(Car_Factory.create(engineProvider));
    }

    @Override
    public void inject(MainActivity activity) {
      injectMainActivity(activity);
    }

    private MainActivity injectMainActivity(MainActivity instance) {
      MainActivity_MembersInjector.injectCar(instance, carProvider.get());
      return instance;
    }
  }
}
```

例如Android中的ApplicationContext，在很多地方都需要注入此依赖，我们可以通过这种方式传入Context。

如果Component使用的Module的构造需要传参，就需要使用在 @Component.Builder 注解的Builder中添加该Module对应的setter方法：
```
@Module
class MyModule(private val name: String) {
    @Provides
    fun provideTire(): Tire {
        return Tire("$name")
    }
}

@Component(modules = [MyModule::class])
interface CarComponent {

    fun inject(activity: MainActivity)

    @Component.Builder
    interface Builder {
        // 必须加上Moduel的setter方法，不然Component无法构造此Module
        fun setMyModule(module: MyModule): Builder

        @BindsInstance
        fun engineName(engineName: String): Builder

        fun build(): CarComponent
    }
}
```
生成的源码中，Component会持有该Module，由外部注入，然后在构造其他需要这个Module提供注入依赖的实例时，就会使用它。而如果是无参构造Module不需要Component的Builder提供setter方法的情况，就直接new实例。相比就是多一步setter，比较简单，不列出源码。

> @Component.Factory 则是生成工厂方法，而不是Builder模式的代码，大同小异。

## 多重绑定
如果我们的一个依赖项是集合类型，并且集合的元素可以是多个依赖提供者，即使属于不同的Module，那我们就要用到多重绑定。Dagger2的多重绑定，可以帮我们把各个依赖提供者提供的实例组装起来，而不用直接依赖各个依赖提供者。

以Dagger2官方文档的Set多重绑定为例：

使用 @IntoSet 标记提供单个元素的 @Provides 方法：
```
@Module
class MyModuleA {
  @Provides @IntoSet
  static String provideOneString(DepA depA, DepB depB) {
    return "ABC";
  }
}
```

也可以使用 @ElementsIntoSet 标记提供多个元素的 @Provides 方法：
```
@Module
class MyModuleB {
  @Provides @ElementsIntoSet
  static Set<String> provideSomeStrings(DepA depA, DepB depB) {
    return new HashSet<String>(Arrays.asList("DEF", "GHI"));
  }
}
```

现在绑定了上面两个Module的Component就可以注入组装了两个依赖提供者的集合了：

```
class Bar {
  @Inject Bar(Set<String> strings) {
    assert strings.contains("ABC");
    assert strings.contains("DEF");
    assert strings.contains("GHI");
  }
}
```
也可以在Component中提供集合：
```
@Component(modules = {MyModuleA.class, MyModuleB.class})
interface MyComponent {
  Set<String> strings();
}

@Test void testMyComponent() {
  MyComponent myComponent = DaggerMyComponent.create();
  assertThat(myComponent.strings()).containsExactly("ABC", "DEF", "GHI");
}
```

> 如果要注入到指定的集合中，@Provides等提供者和需要注入的位置就都要加上自己定义的@Qualifier

Map的多重绑定使用方式，官方文档介绍较清楚，这里不再赘述。

### @Multibinds
@Binds的多重版本，同样是用在Module内的抽象方法上，但是返回类型是集合，比如Map。

## 组件依赖和子组件
在Module部分的介绍中，我们知道Module可以通过 @Module的includes 或者 @Component的modules 来复用，而这一节介绍的是Component的复用和依赖继承关系。如果项目中存在多个需要注入内部字段依赖的对象，相比把所有注入方法都放在同一个Component中，更好的是根据项目模块层次的具体情况，划分为不同的组件。

Component之间可以存在两类关系：
1. 继承关系：使用 @SubComponent
2. 依赖关系：@Component的dependencies属性

### @SubComponent
SubComponent必须关联一个父级Component，从而继承父级Component的依赖注入。以Android中的Application和Activity两个层级作为例子：

```
// 向App注入依赖
@Component(modules = [AppModule::class])
interface AppComponent {

    fun inject(app: App)

    // 获取子组件的Builder
    fun mainActivityComponent(): MainActivityComponent.Builder
}

// 提供App中需要的AppInfo依赖，并声明子组件
@Module(subcomponents = [MainActivityComponent::class])
class AppModule {

    @Provides
    fun getAppInfo(): AppInfo {
        return AppInfo("App 测试")
    }
}

// 子组件，向MainActivity注入依赖
@Subcomponent(modules = [MainActivityModule::class])
interface MainActivityComponent {
    @Subcomponent.Builder
    interface Builder {
        fun build(): MainActivityComponent
    }

    fun inject(activity: MainActivity)
}

// 提供MainActivity中的依赖注入
@Module
class MainActivityModule {

    @Provides
    fun getMainActivityInfo(): MainActivityInfo {
        return MainActivityInfo("MainActivity 测试")
    }
}
```

使用：
```
class App : Application() {
    @Inject
    protected lateinit var appInfo: AppInfo

    lateinit var appComponent: AppComponent
        private set

    override fun onCreate() {
        super.onCreate()
        // AppComponent向App注入AppInfo
        appComponent = DaggerAppComponent.builder().build()
        appComponent.inject(this)
        Log.e("hhhhhhh", "App onCreate: ${appInfo.data}")
    }
}


class MainActivity : AppCompatActivity() {
    @Inject
    protected lateinit var appInfo: AppInfo
    @Inject
    protected lateinit var activityInfo: MainActivityInfo

    override fun onCreate(savedInstanceState: Bundle?) {
        // ......

        // 使用 mainActivityComponent 可以注入appInfo和activityInfo，appInfo来源于父组件AppComponent

        (application as App).appComponent.mainActivityComponent().build().inject(this)
        Log.e("hhhhhhh", "MainActivity onCreate: ${appInfo.data}  ${activityInfo.data}", )
    }
}
```

相关要点：
* SubComponent 会继承 parent Component 的依赖，比如HttpModule中提供的实例，在MainActivityComponent中也能注入，不用AppComponent另外暴露接口。
* SubComponent 只会生成一个parent Component的静态内部私有类，必须通过parent Component结合@Subcomponent.Builder来构建 SubComponent。
* 父子组件的作用域不能相同
* 继承关系生成的源码，子组件会持有父组件对象，通过父组件获取依赖
* 父子组件不能同时提供相同类型的依赖，否则产生冲突，就需要`@Name`或`@Qualifier`来处理

生成代码如下，子组件会持有父组件：
```
public final class DaggerAppComponent {
  private DaggerAppComponent() {
  }

  public static Builder builder() {
    return new Builder();
  }

  public static AppComponent create() {
    return new Builder().build();
  }

  public static final class Builder {
    private AppModule appModule;

    private Builder() {
    }

    public Builder appModule(AppModule appModule) {
      this.appModule = Preconditions.checkNotNull(appModule);
      return this;
    }

    public AppComponent build() {
      if (appModule == null) {
        this.appModule = new AppModule();
      }
      return new AppComponentImpl(appModule);
    }
  }

  private static final class MainActivityComponentBuilder implements MainActivityComponent.Builder {
    private final AppComponentImpl appComponentImpl;

    private MainActivityComponentBuilder(AppComponentImpl appComponentImpl) {
      this.appComponentImpl = appComponentImpl;
    }

    @Override
    public MainActivityComponent build() {
      return new MainActivityComponentImpl(appComponentImpl, new MainActivityModule());
    }
  }

  private static final class MainActivityComponentImpl implements MainActivityComponent {
    private final MainActivityModule mainActivityModule;

    private final AppComponentImpl appComponentImpl;

    private final MainActivityComponentImpl mainActivityComponentImpl = this;

    private MainActivityComponentImpl(AppComponentImpl appComponentImpl,
        MainActivityModule mainActivityModuleParam) {
      this.appComponentImpl = appComponentImpl;
      this.mainActivityModule = mainActivityModuleParam;

    }

    @Override
    public void inject(MainActivity activity) {
      injectMainActivity(activity);
    }

    private MainActivity injectMainActivity(MainActivity instance) {
      // 子组件获取 appInfo 将通过 appComponentImpl
      MainActivity_MembersInjector.injectAppInfo(instance, AppModule_GetAppInfoFactory.getAppInfo(appComponentImpl.appModule));
      MainActivity_MembersInjector.injectActivityInfo(instance, MainActivityModule_GetMainActivityInfoFactory.getMainActivityInfo(mainActivityModule));
      return instance;
    }
  }

  private static final class AppComponentImpl implements AppComponent {
    private final AppModule appModule;

    private final AppComponentImpl appComponentImpl = this;

    private AppComponentImpl(AppModule appModuleParam) {
      this.appModule = appModuleParam;

    }

    @Override
    public void inject(App app) {
      injectApp(app);
    }

    @Override
    public MainActivityComponent.Builder mainActivityComponent() {
      return new MainActivityComponentBuilder(appComponentImpl);
    }

    private App injectApp(App instance) {
      App_MembersInjector.injectAppInfo(instance, AppModule_GetAppInfoFactory.getAppInfo(appModule));
      return instance;
    }
  }
}
```

### @Component的dependencies
与继承关系不同的是，依赖关系的两个Component，不像继承关系那样子组件可以直接拥有父组件提供的依赖。例如组件A依赖组件B，组件A并不是就能直接拥有组件B中的依赖了，而是要在组件B中暴露相关的接口，这样组件A才可以得到组件B中提供的依赖。

还是声明一个提供OkHttpClient实例的Module，这里主要介绍组件依赖关系用法，所以不考虑作用域：
```
// App组件
@Component(modules = [AppModule::class])
interface AppComponent {

    fun inject(app: App)

    // 暴露了提供AppInfo的接口，让依赖App组件的组件可以获取到此对象用于依赖注入
    fun getAppInfo(): AppInfo
}

// App组件使用的Module
@Module
class AppModule {

    @Provides
    fun getAppInfo(): AppInfo {
        return AppInfo("App 测试")
    }
}

// MainActivity组件，依赖了App组件
@Component(modules = [MainActivityModule::class], dependencies = [AppComponent::class])
interface MainActivityComponent {

    fun inject(activity: MainActivity)
}

class MainActivity : AppCompatActivity() {
    @Inject
    protected lateinit var appInfo: AppInfo
    @Inject
    protected lateinit var activityInfo: MainActivityInfo

    override fun onCreate(savedInstanceState: Bundle?) {
        // ......

        // 创建MainActivity组件的时候，需要把App组件实例传入给MainActivity组件的Builder
        DaggerMainActivityComponent.builder().appComponent((application as App).appComponent).build().inject(this)
        Log.e("hhhhhhh", "MainActivity onCreate: ${appInfo.data}  ${activityInfo.data}", )
    }
}
```

生成代码：
```
public final class DaggerMainActivityComponent {
  private DaggerMainActivityComponent() {
  }

  public static Builder builder() {
    return new Builder();
  }

  public static final class Builder {
    private MainActivityModule mainActivityModule;

    private AppComponent appComponent;

    private Builder() {
    }

    public Builder mainActivityModule(MainActivityModule mainActivityModule) {
      this.mainActivityModule = Preconditions.checkNotNull(mainActivityModule);
      return this;
    }

    public Builder appComponent(AppComponent appComponent) {
      this.appComponent = Preconditions.checkNotNull(appComponent);
      return this;
    }

    public MainActivityComponent build() {
      if (mainActivityModule == null) {
        this.mainActivityModule = new MainActivityModule();
      }
      Preconditions.checkBuilderRequirement(appComponent, AppComponent.class);
      return new MainActivityComponentImpl(mainActivityModule, appComponent);
    }
  }

  private static final class MainActivityComponentImpl implements MainActivityComponent {
    private final AppComponent appComponent;

    private final MainActivityModule mainActivityModule;

    private final MainActivityComponentImpl mainActivityComponentImpl = this;

    private MainActivityComponentImpl(MainActivityModule mainActivityModuleParam,
        AppComponent appComponentParam) {
      this.appComponent = appComponentParam;
      this.mainActivityModule = mainActivityModuleParam;

    }

    @Override
    public void inject(MainActivity activity) {
      injectMainActivity(activity);
    }

    private MainActivity injectMainActivity(MainActivity instance) {
      // 注入appInfo时，会调用App组件的getAppInfo()方法来获取
      MainActivity_MembersInjector.injectAppInfo(instance, Preconditions.checkNotNullFromComponent(appComponent.getAppInfo()));
      MainActivity_MembersInjector.injectActivityInfo(instance, MainActivityModule_GetMainActivityInfoFactory.getMainActivityInfo(mainActivityModule));
      return instance;
    }
  }
}
```
和子组件情况不一样，依赖关系的组件生成的代码是分开的，MainActivity组件生成的代码和普通组件一样是一个单独的文件。由于依赖App组件，所以构造MainActivity组件的时候，必须传入App组件对象。

相关要点：
1. 被依赖的Component生命周期更大，作用域不能相同
2. 被依赖的Component必须暴露提供实例的接口，因为需要依赖的Component获取实例，是用被依赖的Componnet来提供实例。暴露的依赖不能多层传递

## 辅助注入
每次构造时都要自行传入参数的情况，可以使用辅助注入：
```
class MyDataService @AssistedInject constructor(
    dataFetcher: DataFetcher, // 自动注入
    @Assisted config: Config // 使用 @Assisted 注解的参数由用户注入
)

// 用于构造 MyDataService，可以传入参数
@AssistedFactory
interface MyDataServiceFactory {
  fun create(config: Config): MyDataService
}
```

使用方式：
```
// Dagger2的方式注入（如果使用Hilt则不用声明Componennt直接注入即可）
@Component
interface MyComponent {
    fun createDataServiceFactory(): MyDataServiceFactory
}

class MyApp {
  private val serviceFactory = DaggerMainComponent.builder().build().createDataServiceFactory()

  fun setupService(config: Config): MyDataService {
    val service = serviceFactory.create(config)
    ...
    return service
  }
}
```

如果使用Dagger2的自定义Componennt，则在Component实现类中将持有`Provider<MyDataServiceFactory>`对象，用于提供`MyDataServiceFactory`对象。如果使用Hilt则会根据注入位置在例如ActivityCImpl等类的内部持有`Provider<MyDataServiceFactory>`，用于注入`MyDataServiceFactory`对象。

> 辅助注入不能使用`@Scope`，所以每次生成不同的实例。

## 异步依赖注入
使用较少，且官方文档描述较清晰，这里不展开介绍。 [Dagger2 producers](https://dagger.dev/dev-guide/producers)

## Android
Dagger如果能够创建所有注入的对象时效果最佳，但是很多Android中框架类都是由OS实例化的，例如 Activity和Fragment。您必须在生命周期方法中执行成员注入，这意味着许多类最终看起来像这样：
```
public class FrombulationActivity extends Activity {
  @Inject Frombulator frombulator;

  @Override
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    // DO THIS FIRST. Otherwise frombulator might be null!
    ((SomeApplicationBaseType) getContext().getApplicationContext())
        .getApplicationComponent()
        .newActivityComponentBuilder()
        .activity(this)
        .build()
        .inject(this);
    // ... now you can write the exciting code
  }
}
```
这主要存在两个问题 ：
1. 复制粘贴代码使以后难以重构。随着越来越多的开发人员复制粘贴该块，越来越少的人会知道它的实际作用。
2. 从根本上讲，它要求请求注入（FrombulationActivity）的类型知道其注入器。即使这是通过接口而不是具体类型完成的，也打破了依赖注入的核心原理：类不应该知道如何注入依赖。

使用普通的Dagger2来完成Android中的依赖注入，可以参考[官方文档](https://developer.android.google.cn/training/dependency-injection/dagger-android)

还有一种选择是使用 dagger.android 提供简化以上问题的方式，使用它需要学习额外的API和概念。

https://www.jianshu.com/p/8060a260488d