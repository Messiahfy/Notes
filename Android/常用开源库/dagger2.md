https://developer.android.google.cn/training/dependency-injection/dagger-android#kotlin
https://codelabs.developers.google.com/codelabs/android-dagger/#6
https://dagger.dev/dev-guide/android
https://www.jianshu.com/p/8060a260488d

## 概述
Dagger2是一个依赖注入框架，它会根据开发者使用相关注解写的依赖关系的代码，在编译时生成相关代码，然后开发者可以使用生成的代码完成依赖注入。Dagger2中使用的注解，一部分是属于javax.inject的，例如@Inject；另一部分则是属于Dagger2自行创造的，例如@Component。属于javax.inject的那一部分注解，使用方式须按照Dagger2的要求，所以可能会和javax.inject文档中的使用说明有差异。

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
public final class DaggerCarComponent implements CarComponent {
  private DaggerCarComponent(Builder builder) {}

  public static Builder builder() {
    return new Builder();
  }

  public static CarComponent create() {
    return new Builder().build();
  }

  @Override
  public Car createCar() {
    return new Car(new Engine());
  }

  public static final class Builder {
    private Builder() {}

    public CarComponent build() {
      return new DaggerCarComponent(this);
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
    Car car;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        DaggerCarComponent.create().inject(this);
        car.start();
    }
}
```
MainActivity类中用 `@Inject` 修饰了 Car 字段，然后在使用生成的代码后，car 字段就被赋值了，然后就可以开始使用。

通过生成的代码，可以看到 inject 方法就是给 MainActivity 实例中的 car 字段赋值，这里省略具体生成的代码。

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
用 @Inject 修饰setEngine2方法，告知 Dagger2 构造 Car 时，要调用setEngine2方法，传入Engine2实例，Engine2的实例也有Dagger2构造。方法注入和其他注入方式可以一起使用，当然，需要各自注入各自的实例，如果注入相同实例，那么就会注入两次，后面注入的会覆盖前面注入的实例。

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

> 如果同时存在 @Module+@Provides的方式提供实例 和 @Inject构造函数，Dagger2内部会优先选择@Module+@Provides提供的实例。

Module其实就相当于一个获取实例的工厂

**@Provides修饰的方法也可以包含参数，Dagger2会寻找本Module内返回为该参数类型且被@Provides修饰的方法，如果没有，就寻找该类是否有被@Inject修饰的构造函数**

> @Module注解可以使用 includes 属性，作用就是复用Module，相当于把包含的一个或多个Module中提供的实例获取方式都组合到了当前Module。这个包含动作是递归的，包含的Module又包含了其他Module，则都会组合到当前Module。<br/>
也可以@Component的modules数组包含多个Module
-----------------------

前面对实例的依赖关系都是用具体的类型来决定的，但是我们开发中很多时候都是面向接口编程，实例的类型都是接口，这种情况下，我们可以也通过 @Provides 来完成依赖注入：
```
public class Car {
    private IEngine engine;

    @Inject
    public Car(IEngine engine) {
        this.engine = engine;
    }


    public void start() {
        engine.start();
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

## 3. @Binds
@Binds和@Provides的作用类似，但是要求Module是抽象类，修饰的方法是抽象方法。比@Provides可以简化一步 return 过程。
```
@Module
abstract class CarModule {

    @Binds
    abstract IEngine getEngine(Engine engine);
}
```

## 4. @Qualifier 和 @Named
前面的依赖注入方式，都是以类型来确定注入依赖的构造方式。但是在一些情况下，构造一个类型的实例的方式有多种，比如两个@Provides修饰的方法返回相同类型的不同实例，此时Dagger2就无法知道选择哪个方法来得到实例。这种情况下，我们可以通过 @Qualifier 或者 @Name 来确定选择哪个。

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
        Car car1 = carProvider.get());
        Car car2 = carProvider.get());
        Car car3 = carProvider.get());
    }
}
```

## 4. 作用域
默认情况下，Component 每次调用 Module 中的 provide 方法或 Inject 构造函数时都会创建一个新的实例，如果我们需要单例，就要用到作用域注解。作用域机制可以保证在`@Scope`标记的`Component`作用域内，类为单例。

@Singleton是javax.inject自带的作用域注解，这个名称没有实际意义，我们也可以用@Scope自定义任何名称的作用域注解。@Singleton的源码如下：
```
@Scope
@Documented
@Retention(RUNTIME)
public @interface Singleton {}
```

作用域注解只能标注目标类（被@Inject标注构造函数的类）、@provides（@Binds） 方法和 Component。也就是说，作用域注解要生效的话，需要同时标注在 Component 和依赖注入实例的提供者。

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


注意点：
1. 依赖注入实例的提供者的作用域注解必须和绑定的 Component 的 作用域注解一样
2. Component 可以同时被多个作用域注解标记，也就是说 Component 可以和多个不同的作用域的依赖注入实例的提供者绑定使用。
3. component的dependencies与component自身的scope不能相同：
4. Singleton的组件不能依赖其他的scope的组件，只能其他scope的组件依赖Singleton的组件。所以@Singleton一般用于全局级别的单例 ???
5. 没有scope的component不能依赖有scope的component

作用域注解后的单例都是局部单例，仅在同一个Component实例范围内是单例，在生成的DaggerXxxComponent中，会用同一个DoubleCheck来获取对象。要实现全局单例，就要用同一个Component，比如在Application中缓存一个Component，然后都用这个Component。

## 5. @Reusable 
@Reusable用于标记依赖注入的提供者，它提供的复用，不是绝对的同步单例，而是一个“尽量复用”逻辑。**不需要**Component中也使用此注解。

## 6. 可选绑定 @BindsOptionalOf
```
@BindsOptionalOf abstract Car optionalCar();
```
在Module中使用 @BindsOptionalOf，在需要注入Car的地方，类型可以使用：
* Optional<Car>
* Optional<Provider<Car>>
* Optional<Lazy<Car>>
* Optional<Provider<Lazy<Car>>>
如果，组件中可以找到 Car 实例的提供者，如@Inject修饰的Car的构造函数，或者Module中的@Provides等方式，那么可以通过Optional得到 Car；如果找不到，那么通过 Optional 得到的Car就是null。


## 7. @BindsInstance
如果Component中的依赖项或者其他参数需要从外部传入，我们可以用Component的Builder传入自行构造Module的方式；同样的目的，我们也可以使用@BindsInstance。

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
        Builder engineName( String engineName);

        CarComponent build();
    }
}
```
现在生成的Component代码中不会再有create()方法，必须使用Builder来构造CarComponent，此时Builder会有一个engineName(String)方法用于传入该参数:
```
DaggerCarComponent.builder().engineName("传入的引擎名称").build().inject(this);
```

例如Android中的ApplicationContext，在很多地方都需要注入此依赖，我们可以通过这种方式传入Context。

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
把用于注入MainActivity内部依赖的Component作为SubComponent，其中需要声明一个@Subcomponent.Builder修饰的Builder：
```
@Subcomponent
public interface MainActivityComponent {

    @Subcomponent.Builder
    interface Builder {
        MainActivityComponent build();
    }

    void inject(MainActivity activity);
}
```

然后Module的subcomponents包含此SubComponent：
```
@Module(subcomponents = {MainActivityComponent.class})
public class SubComponentsModule {
}
```

现在就在AppComponent中绑定SubComponentsModule，并且要写出获取MainActivityComponent.Builder的方法，对应到Subcomponent中写的Builder。可以看出，SubComponentsModule就相当于Subcomponent和ParentComponent的连接者。
```
@Component(modules = {HttpModule.class, SubComponentsModule.class})
public interface AppComponent {
    MainActivityComponent.Builder mainActivityComponent();
}
```

现在就可以在MainActivity中通过AppComponent得到mainActivityComponent，然后注入依赖。
```
DaggerAppComponent.create().mainActivityComponent().build().inject(this);
```

**虽然官方文档的示例如上使用了类似SubComponentsModule这个中转，但是验证发现不用这个中转也可以实现组件继承，并且@Module的subcomponents属性标记的@Beta**

相关要点：
1. SubComponent 会继承 parent Component 的依赖，比如HttpModule中提供的实例，在MainActivityComponent中也能注入，不用AppComponent另外暴露接口。
2. SubComponent 只会生成一个parent Component的内部私有类，必须通过parent Component结合@Subcomponent.Builder来构建 SubComponent。
3. 父子组件的作用域不能相同
4. 继承关系生成的源码，子组件获取依赖就是直接使用父组件的Module

### @Component的dependencies
与继承关系不同的是，依赖关系的两个Component，他们可以提供的依赖还是各自分开的，不像继承关系那样子组件可以直接拥有父组件提供的依赖。例如组件A依赖组件B，组件A并不是就能直接拥有组件B中的依赖了，而是要在组件B中暴露相关的接口，这样组件A才可以得到组件B中提供的依赖。

还是声明一个提供OkHttpClient实例的Module，这里主要介绍组件依赖关系用法，所以不考虑作用域：
```
@Module
public class HttpModule {
    @Provides
    OkHttpClient provideOkHttpClient() {
        return new OkHttpClient.Builder().build();
    }
}
```

然后同样是一个绑定HttpModule的HttpComponent，并且暴露出提供OkHttpClient的接口：
```
@Component(modules = {HttpModule.class})
public interface HttpComponent {
    OkHttpClient createHttpClient();
}
```

声明MyComponent，并且依赖HttpComponent：
```
@Component(dependencies = HttpComponent.class)
public interface MyComponent {
    void inject(MainActivity activity);
}
```

完成以上步骤，我们就可以在MainActivity中使用：
```
HttpComponent httpComponent = DaggerHttpComponent.builder().build();
DaggerMyComponent.builder().httpComponent(httpComponent).build().inject(this);
```
可以看到，构造MyComponent必须传入httpComponent，因为依赖关系生成的源码中，MyComponent注入依赖就是调用httpComponent的createHttpClient()方法。


相关要点：
1. 被依赖的Component生命周期更大，作用域不能相同
2. 被依赖的Component必须暴露提供实例的接口，因为需要依赖的Component获取实例，是用被依赖的Componnet来提供实例。暴露的依赖不能多层传递

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