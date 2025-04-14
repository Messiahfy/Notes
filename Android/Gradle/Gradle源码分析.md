以 Gradle 8.7 版本 为例


groovy脚本加载流程：[Gradle深入解析- Groovy Script加载流程()](https://juejin.cn/post/7219925045228240954)

kotlin脚本加载流程：[Gradle深入解析- Kotlin Script加载流程](https://juejin.cn/post/7219988638050369594)

https://juejin.cn/post/6972061473149452296
https://sharrychoo.github.io/blog/gradle-source/launch
https://www.jianshu.com/p/d1099b77a753
https://www.jianshu.com/p/625bc82003d7


https://github.com/gradle/gradle/blob/v8.7.0/platforms/core-runtime/wrapper/src/main/java/org/gradle/wrapper/GradleWrapperMain.java

GradleWrappe启动

获取了 gradle-wrapper 相关的文件
解析命令行参数
便调用了 WrapperExecutor.execute 继续执行后续的任务


WrapperExecutor 的 execute 主要负责找寻对应的 gradle 版本, 然后委托给 BootstrapMainStarter 执行后续操作

// 从 gradle home 中找寻 gradle 的 jar 包

// 获取 gradle 的入口函数所在类 GradleMain
        Class<?> mainClass = contextClassLoader.loadClass("org.gradle.launcher.GradleMain");


https://github.com/gradle/gradle/blob/v8.7.0/platforms/core-runtime/bootstrap/src/main/java/org/gradle/launcher/GradleMain.java

Class<?> mainClass = Class.forName("org.gradle.launcher.bootstrap.ProcessBootstrap");
        Method mainMethod = mainClass.getMethod("run", String.class, String[].class);
        mainMethod.invoke(null, "org.gradle.launcher.Main", args);


https://github.com/gradle/gradle/blob/v8.7.0/platforms/core-runtime/launcher/src/main/java/org/gradle/launcher/bootstrap/ProcessBootstrap.java


https://github.com/gradle/gradle/blob/v8.7.0/platforms/core-runtime/launcher/src/main/java/org/gradle/launcher/Main.java

Main的父类EntryPoint的run方法会执行doAction

```
/**
 * The main command-line entry-point for Gradle.
 */
public class Main extends EntryPoint {
    public static void main(String[] args) {
        new Main().run(args);
    }

    @Override
    protected void doAction(String[] args, ExecutionListener listener) {
        createActionFactory().convert(Arrays.asList(args)).execute(listener);
    }

    CommandLineActionFactory createActionFactory() {
        return new DefaultCommandLineActionFactory();
    }
}
```

https://github.com/gradle/gradle/blob/v8.7.0/platforms/core-runtime/launcher/src/main/java/org/gradle/launcher/cli/DefaultCommandLineActionFactory.java

```
public class DefaultCommandLineActionFactory implements CommandLineActionFactory {
    public static final String WELCOME_MESSAGE_ENABLED_SYSTEM_PROPERTY = "org.gradle.internal.launcher.welcomeMessageEnabled";
    private static final String HELP = "h";
    private static final String VERSION = "v";
    private static final String VERSION_CONTINUE = "V";

    /**
     * <p>Converts the given command-line arguments to an {@link Action} which performs the action requested by the
     * command-line args.
     *
     * @param args The command-line arguments.
     * @return The action to execute.
     */
    @Override
    public CommandLineExecution convert(List<String> args) {
        ServiceRegistry loggingServices = createLoggingServices();

        LoggingConfiguration loggingConfiguration = new DefaultLoggingConfiguration();

        return new WithLogging(loggingServices,
            args,
            loggingConfiguration,
            new ParseAndBuildAction(loggingServices, args),
            new BuildExceptionReporter(loggingServices.get(StyledTextOutputFactory.class), loggingConfiguration, clientMetaData()));
    }

    private static GradleLauncherMetaData clientMetaData() {
        return new GradleLauncherMetaData();
    }

    // ......
}
```


```
private static class WithLogging implements CommandLineExecution {
    private final ServiceRegistry loggingServices;
    private final List<String> args;
    private final LoggingConfiguration loggingConfiguration;
    private final Action<ExecutionListener> action;
    private final Action<Throwable> reporter;
    WithLogging(ServiceRegistry loggingServices, List<String> args, LoggingConfiguration loggingConfiguration, Action<ExecutionListener> action, Action<Throwable> reporter) {
        this.loggingServices = loggingServices;
        this.args = args;
        this.loggingConfiguration = loggingConfiguration;
        this.action = action;
        this.reporter = reporter;
    }
    @Override
    public void execute(ExecutionListener executionListener) {
        BuildOptionBackedConverter<WelcomeMessageConfiguration> welcomeMessageConverter = new BuildOptionBackedConverter<>(new WelcomeMessageBuildOptions());
        BuildOptionBackedConverter<LoggingConfiguration> loggingBuildOptions = new BuildOptionBackedConverter<>(new LoggingConfigurationBuildOptions());
        InitialPropertiesConverter propertiesConverter = new InitialPropertiesConverter();
        BuildLayoutConverter buildLayoutConverter = new BuildLayoutConverter();
        LayoutToPropertiesConverter layoutToPropertiesConverter = new LayoutToPropertiesConverter(new BuildLayoutFactory());

        BuildLayoutResult buildLayout = buildLayoutConverter.defaultValues();

        CommandLineParser parser = new CommandLineParser();
        propertiesConverter.configure(parser);
        buildLayoutConverter.configure(parser);
        loggingBuildOptions.configure(parser);

        parser.allowUnknownOptions();
        parser.allowMixedSubcommandsAndOptions();

        WelcomeMessageConfiguration welcomeMessageConfiguration = new WelcomeMessageConfiguration(WelcomeMessageDisplayMode.ONCE);

        try {
            ParsedCommandLine parsedCommandLine = parser.parse(args);
            InitialProperties initialProperties = propertiesConverter.convert(parsedCommandLine);

            // Calculate build layout, for loading properties and other logging configuration
            buildLayout = buildLayoutConverter.convert(initialProperties, parsedCommandLine, null);

            // Read *.properties files
            AllProperties properties = layoutToPropertiesConverter.convert(initialProperties, buildLayout);

            // Calculate the logging configuration
            loggingBuildOptions.convert(parsedCommandLine, properties, loggingConfiguration);

            // Get configuration for showing the welcome message
            welcomeMessageConverter.convert(parsedCommandLine, properties, welcomeMessageConfiguration);
        } catch (CommandLineArgumentException e) {
            // Ignore, deal with this problem later
        }

        LoggingManagerInternal loggingManager = loggingServices.getFactory(LoggingManagerInternal.class).create();
        loggingManager.setLevelInternal(loggingConfiguration.getLogLevel());
        loggingManager.start();
        try {
            Action<ExecutionListener> exceptionReportingAction =
                new ExceptionReportingAction(reporter, loggingManager,
                    new NativeServicesInitializingAction(buildLayout, loggingConfiguration, loggingManager,
                        new WelcomeMessageAction(buildLayout, welcomeMessageConfiguration,
                            new DebugLoggerWarningAction(loggingConfiguration, action))));
            exceptionReportingAction.execute(executionListener);
        } finally {
            loggingManager.stop();
        }
    }
}
```
执行 exceptionReportingAction execute，实际会执行 action 也就是 ParseAndBuildAction

```
private class ParseAndBuildAction extends NonParserConfiguringCommandLineActionCreator implements Action<ExecutionListener> {
        private final ServiceRegistry loggingServices;
        private final List<String> args;
        private List<CommandLineActionCreator> actionCreators;
        private CommandLineParser parser = new CommandLineParser();

        private ParseAndBuildAction(ServiceRegistry loggingServices, List<String> args) {
            this.loggingServices = loggingServices;
            this.args = args;

            actionCreators = new ArrayList<>();
            actionCreators.add(new BuiltInActionCreator());
            actionCreators.add(new ContinuingActionCreator());
        }

        @Override
        public void execute(ExecutionListener executionListener) {
            // This must be added only during execute, because the actual constructor is called by various tests and this will not succeed if called then
            createBuildActionFactoryActionCreator(loggingServices, actionCreators);
            configureCreators();

            Action<? super ExecutionListener> action;
            try {
                ParsedCommandLine commandLine = parser.parse(args);
                action = createAction(parser, commandLine);
            } catch (CommandLineArgumentException e) {
                action = new CommandLineParseFailureAction(parser, e);
            }

            action.execute(executionListener);
        }

        private void configureCreators() {
            actionCreators.forEach(creator -> creator.configureCommandLineParser(parser));
        }

        @Override
        public Action<? super ExecutionListener> createAction(CommandLineParser parser, ParsedCommandLine commandLine) {
            List<Action<? super ExecutionListener>> actions = new ArrayList<>(2);
            for (CommandLineActionCreator actionCreator : actionCreators) {
                Action<? super ExecutionListener> action = actionCreator.createAction(parser, commandLine);
                if (action != null) {
                    actions.add(action);
                    if (!(action instanceof ContinuingAction)) {
                        break;
                    }
                }
            }

            if (!actions.isEmpty()) {
                return Actions.composite(actions);
            }

            throw new UnsupportedOperationException("No action factory for specified command-line arguments.");
        }
    }
```
收集可以创建 action 的 Factory
```
protected void createBuildActionFactoryActionCreator(ServiceRegistry loggingServices, List<CommandLineActionCreator> actionCreators) {
        actionCreators.add(new BuildActionsFactory(loggingServices));
    }
```

这里只需要关注 BuildActionsFactory

```
        private void configureCreators() {
            actionCreators.forEach(creator -> creator.configureCommandLineParser(parser));
        }
```
通过 BuildActionsFactory 创建一个 action

```
public Action<? super ExecutionListener> createAction(CommandLineParser parser, ParsedCommandLine commandLine) {
    List<Action<? super ExecutionListener>> actions = new ArrayList<>(2);
    for (CommandLineActionCreator actionCreator : actionCreators) {
        Action<? super ExecutionListener> action = actionCreator.createAction(parser, commandLine);
        if (action != null) {
            actions.add(action);
            if (!(action instanceof ContinuingAction)) {
                break;
            }
        }
    }

    if (!actions.isEmpty()) {
        return Actions.composite(actions);
    }

    throw new UnsupportedOperationException("No action factory for specified command-line arguments.");
}
```
委托给这个 action 执行后续的启动任务


```
class BuildActionsFactory implements CommandLineActionCreator {
    private final ParametersConverter parametersConverter;
    private final ServiceRegistry loggingServices;
    private final JvmVersionDetector jvmVersionDetector;
    private final FileCollectionFactory fileCollectionFactory;
    private final ServiceRegistry basicServices;

    public BuildActionsFactory(ServiceRegistry loggingServices) {
        basicServices = ServiceRegistryBuilder.builder()
            .scope(Scope.Global.class)
            .displayName("Basic global services")
            .parent(loggingServices)
            .parent(NativeServices.getInstance())
            .provider(new BasicGlobalScopeServices())
            .build();
        this.loggingServices = loggingServices;
        fileCollectionFactory = basicServices.get(FileCollectionFactory.class);
        parametersConverter = new ParametersConverter(new BuildLayoutFactory(), basicServices.get(FileCollectionFactory.class));
        jvmVersionDetector = basicServices.get(JvmVersionDetector.class);
    }

    @Override
    public void configureCommandLineParser(CommandLineParser parser) {
        parametersConverter.configure(parser);
    }

    @Override
    public Action<? super ExecutionListener> createAction(CommandLineParser parser, ParsedCommandLine commandLine) {
        Parameters parameters = parametersConverter.convert(commandLine, null);

        parameters.getDaemonParameters().applyDefaultsFor(jvmVersionDetector.getJavaVersion(parameters.getDaemonParameters().getEffectiveJvm()));

        if (parameters.getDaemonParameters().isStop()) {
            return Actions.toAction(stopAllDaemons(parameters.getDaemonParameters()));
        }
        if (parameters.getDaemonParameters().isStatus()) {
            return Actions.toAction(showDaemonStatus(parameters.getDaemonParameters()));
        }
        if (parameters.getDaemonParameters().isForeground()) {
            DaemonParameters daemonParameters = parameters.getDaemonParameters();
            ForegroundDaemonConfiguration conf = new ForegroundDaemonConfiguration(
                UUID.randomUUID().toString(), daemonParameters.getBaseDir(), daemonParameters.getIdleTimeout(), daemonParameters.getPeriodicCheckInterval(), fileCollectionFactory,
                daemonParameters.shouldApplyInstrumentationAgent());
            return Actions.toAction(new ForegroundDaemonAction(loggingServices, conf));
        }
        if (parameters.getDaemonParameters().isEnabled()) {
            return Actions.toAction(runBuildWithDaemon(parameters.getStartParameter(), parameters.getDaemonParameters()));
        }
        if (canUseCurrentProcess(parameters.getDaemonParameters())) {
            return Actions.toAction(runBuildInProcess(parameters.getStartParameter(), parameters.getDaemonParameters()));
        }

        return Actions.toAction(runBuildInSingleUseDaemon(parameters.getStartParameter(), parameters.getDaemonParameters()));
    }

    //.....

    private Runnable runBuildInProcess(StartParameterInternal startParameter, DaemonParameters daemonParameters) {
        ServiceRegistry globalServices = ServiceRegistryBuilder.builder()
            .scope(Scope.Global.class)
            .displayName("Global services")
            .parent(loggingServices)
            .parent(NativeServices.getInstance())
            .provider(new GlobalScopeServices(startParameter.isContinuous(), AgentStatus.of(daemonParameters.shouldApplyInstrumentationAgent())))
            .build();

        globalServices.get(AgentInitializer.class).maybeConfigureInstrumentationAgent();

        // Force the user home services to be stopped first, the dependencies between the user home services and the global services are not preserved currently
        return runBuildAndCloseServices(startParameter, daemonParameters, globalServices.get(BuildExecuter.class), globalServices, globalServices.get(GradleUserHomeScopeServiceRegistry.class));
    }

    private Runnable runBuildAndCloseServices(StartParameterInternal startParameter, DaemonParameters daemonParameters, BuildActionExecuter<BuildActionParameters, BuildRequestContext> executer, ServiceRegistry sharedServices, Object... stopBeforeSharedServices) {
        BuildActionParameters parameters = createBuildActionParameters(startParameter, daemonParameters);
        Stoppable stoppable = new CompositeStoppable().add(stopBeforeSharedServices).add(sharedServices);
        return new RunBuildAction(executer, startParameter, clientMetaData(), getBuildStartTime(), parameters, sharedServices, stoppable);
    }

}
```

```
public class ServiceRegistryBuilder {
    private final List<ServiceRegistry> parents = new ArrayList<ServiceRegistry>();
    private final List<Object> providers = new ArrayList<Object>();
    private String displayName;
    private Class<? extends Scope> scope;

    private ServiceRegistryBuilder() {
    }

    public static ServiceRegistryBuilder builder() {
        return new ServiceRegistryBuilder();
    }

    public ServiceRegistryBuilder displayName(String displayName) {
        this.displayName = displayName;
        return this;
    }

    public ServiceRegistryBuilder parent(ServiceRegistry parent) {
        this.parents.add(parent);
        return this;
    }

    public ServiceRegistryBuilder provider(Object provider) {
        this.providers.add(provider);
        return this;
    }

    /**
     * Providing a scope makes the resulting {@link ServiceRegistry}
     * validate all registered services for being annotated with the given scope.
     */
    public ServiceRegistryBuilder scope(Class<? extends Scope> scope) {
        this.scope = scope;
        return this;
    }

    public ServiceRegistry build() {
        ServiceRegistry[] parents = this.parents.toArray(new ServiceRegistry[0]);

        // 创建服务注册器 ScopedServiceRegistry 或 DefaultServiceRegistry
        DefaultServiceRegistry registry = scope != null
            ? new ScopedServiceRegistry(scope, displayName, parents)
            : new DefaultServiceRegistry(displayName, parents);

        // 为服务注册器添加 GlobalScopeServices
        for (Object provider : providers) {
            registry.addProvider(provider);
        }
        return registry;
    }
}
```



```
public class RunBuildAction implements Runnable {
    private final BuildActionExecuter<BuildActionParameters, BuildRequestContext> executer;
    private final StartParameterInternal startParameter;
    private final BuildClientMetaData clientMetaData;
    private final long startTime;
    private final BuildActionParameters buildActionParameters;
    private final ServiceRegistry sharedServices;
    private final Stoppable stoppable;

    public RunBuildAction(BuildActionExecuter<BuildActionParameters, BuildRequestContext> executer, StartParameterInternal startParameter, BuildClientMetaData clientMetaData, long startTime,
                          BuildActionParameters buildActionParameters, ServiceRegistry sharedServices, Stoppable stoppable) {
        this.executer = executer;
        this.startParameter = startParameter;
        this.clientMetaData = clientMetaData;
        this.startTime = startTime;
        this.buildActionParameters = buildActionParameters;
        this.sharedServices = sharedServices;
        this.stoppable = stoppable;
    }

    @Override
    public void run() {
        try {
            BuildActionResult result = executer.execute(
                new ExecuteBuildAction(startParameter),
                buildActionParameters,
                new DefaultBuildRequestContext(new DefaultBuildRequestMetaData(clientMetaData, startTime, sharedServices.get(ConsoleDetector.class).isConsoleInput()), new DefaultBuildCancellationToken(), new NoOpBuildEventConsumer())
            );
            if (result.hasFailure()) {
                // Don't need to unpack the serialized failure. It will already have been reported and is not used by anything downstream of this action.
                throw new ReportedException();
            }
        } finally {
            stoppable.stop();
        }
    }
}
```