### 一、SpringBoot 内部操作

#### 1、创建 SpringApplication 对象

a）设置 primarySources、webApplicationType 和 mainApplicationClass

b）扫描 classpath 下面所有的 **META-INF/spring.factories** 文件，这些文件应该保存有 spring 的接口和这些接口的实现类，这样就可以在发布 jar 包的时候配置一些 Initializer 或者 Listener 来达到定制化 spring 的目的。这是 SPI 的一种实现。

c）实例化并排序 **ApplicationContextInitializer** 和 **ApplicationListener** 的实现，设置到 SpringApplication 里面。这两个接口都是 spring 框架里面的而不是 spring boot 里面的。



#### 2、调 run 方法运行 SpringApplication

a）实例化并排序构造器为 (SpringApplication, String[]) 的 **SpringApplicationRunListener** 类。

​		在 spring boot 初始化的每个阶段，都会回调这个接口相应的方法。只有一个实现类 **EventPublishingRunListener**，contextLoaded 及之前，通过自己的广播器广播事件；contextLoaded 时把 ApplicationListener 设置到 context，并在之后的时间中通过 context 广播事件。

b）**starting()** 对应 **ApplicationStartingEvent**。

c）调 prepareEnvironment 方法根据 webApplicationType  准备 Environment。

​		构造方法内加载 systemProperties 和 systemEnvironment，创建 servletConfigInitParams 和 servletContextInitParams 占位资源。为 Environment 对象设置 ConversionService、defaultProperties、PropertySources（包括 program arguments 的参数）和 Profiles（包括 VM options 的设置）。**environmentPrepared()** 对应 **ApplicationEnvironmentPreparedEvent**。**ConfigFileApplicationListener** 同时也是一个 EnvironmentPostProcessor，会去解析**配置文件**。

d）调 createApplicationContext 方法根据 webApplicationType  创建 ConfigurableApplicationContext 对象。

​		在构造方法内，先创建 AnnotatedBeanDefinitionReader 对象，注册 **ConfigurationClassPostProcessor**、AutowiredAnnotationBeanPostProcessor、CommonAnnotationBeanPostProcessor、PersistenceAnnotationBeanPostProcessor、EventListenerMethodProcessor 和 DefaultEventListenerFactory 这些 BeanDefinition。

​		然后创建 ClassPathBeanDefinitionScanner 对象，添加 Component、ManagedBean 和 Named 注解为默认过滤器，加载 META-INF/spring.components 中的资源。

e）通过 FailureAnalyzers 类，加载所有 FailureAnalyzer 接口的实现类并实例化和排序，然后把 BeanFactory 和 Environment 设置到对应的对象中。这些对象能够分析错误错误并返回优先级最高的结果。必要的时候会加载 FailureAnalysisReporter 对象来报告错误。

f）调 prepareContext 准备 context。

​		**ApplicationContextInitializer.initialize()**。DelegatingApplicationContextInitializer 是一个装饰器，会读取 env 中的 context.initializer.classes 数据，加载这个变量指定的 ApplicationContextInitializer 并回调其 initialize() 方法。

​		**contextPrepared()** 对应 **ApplicationContextInitializedEvent**。向 BeanFactory 注册 springApplicationArguments 和 springBootBanner 单例 bean，设置 allowBeanDefinitionOverriding。加载主类和其他 source 类的定义。**contextLoaded()** 对应 **ApplicationPreparedEvent**。

g）调 refreshContext 方法刷新 context。

​		这里就走到 spring 框架的逻辑了，先调 AbstractApplicationContext.refresh 方法，这个在**[下文](#refresh)**详细分析。然后注册关闭时的钩子，向订阅了 ContextClosedEvent 事件的 ApplicationListener 发送事件，释放各种资源。

h）**started()** 对应 **ApplicationStartedEvent**。

i）回调 ApplicationRunner.run() 和 CommandLineRunner.run()

j）**running()** 对应 **ApplicationReadyEvent**。



### 二、Spring Framework 内部操作

#### <span id="refresh">1、调 AbstractApplicationContext.refresh() 方法</span>

a）获取 startupShutdownMonitor 锁，以防多个线程同时调用这个方法。

b）调 prepareRefresh 方法准备刷新 context。earlyApplicationListeners 存在则覆盖 applicationListeners，否则相反。

c）调 prepareBeanFactory 初始化 BeanFactory。

​		注册 BeanPostProcessor：ApplicationContextAwareProcessor、ApplicationListenerDetector。

​		忽略 Interface：EnvironmentAware、EmbeddedValueResolverAware、ResourceLoaderAware、ApplicationEventPublisherAware、MessageSourceAware、ApplicationContextAware。

​		注册 ResolvableDependency：BeanFactory、ResourceLoader、ApplicationEventPublisher、ApplicationContext。

​		注册 Singleton：environment、systemProperties、systemEnvironment。

d）调 postProcessBeanFactory 方法

​		注册 BeanPostProcessor：WebApplicationContextServletContextAwareProcessor

​		忽略 Interface：ServletContextAware

​		注册 ResolvableDependency：ServletRequest、ServletResponse、HttpSession、WebRequest

​		注册 Scope：request、session

e）调 invokeBeanFactoryPostProcessors 方法。

​		SharedMetadataReaderFactoryContextInitializer 注册 CachingMetadataReaderFactoryPostProcessor

​		ConfigurationWarningsApplicationContextInitializer 注册 ConfigurationWarningsPostProcessor

​		ConfigFileApplicationListener 注册 PropertySourceOrderingPostProcessor

​		先处理以上几个 **BeanFactoryPostProcessor** 中的 **BeanDefinitionRegistryPostProcessor** 。

​		按预先定义、PriorityOrdered、Ordered 和其他的顺序，回调 **BeanDefinitionRegistryPostProcessor** 的 postProcessBeanDefinitionRegistry 方法，**ConfigurationClassPostProcessor** 类会**加载所有的 bean definition**。然后回调 postProcessBeanFactory 方法。这个接口允许用户在 BeanFactory  初始化之后，对 BeanDefinition 进行操作。

​		然后按顺序回调 **BeanFactoryPostProcessor** 的 postProcessBeanFactory  方法。

f）调 registerBeanPostProcessors 方法。

​		添加 BeanPostProcessorChecker 为 BeanPostProcessor，添加所有其他 BeanPostProcessor，添加 ApplicationListenerDetector 为 BeanPostProcessor。

g）调 initMessageSource 方法。注册 messageSource 的单例 bean。利用装饰器模式，装饰 parent context 的 messageSource，本身不做任何操作。

h）调 initApplicationEventMulticaster 方法。注册 applicationEventMulticaster 的单例 bean。

i）调 onRefresh 方法。创建并启动 webServer，但此时还没有 Connector，因此不会监听端口。

j）调 registerListeners 方法。

​		把所有 ApplicationListener 对象放到广播器。添加 ApplicationListener 的 beanName 到广播器。如果有 earlyApplicationEvents 事件，则向 ApplicationListener  广播这些事件。

k）调 finishBeanFactoryInitialization 方法。**这一步会完成所有对象的实例化。**

l）调 finishRefresh 方法，完成整个 context 的创建。

​		注册 lifecycleProcessor 的单例 bean。回调 Lifecycle 的  start() 方法。广播 ContextRefreshedEvent 事件。注册 LiveBeansView 信息（看注释是给 Spring Tool Suite 使用的）。

​		回调 webServer 的 start() 方法，此时服务器可以提供服务了。

​		广播 ServletWebServerInitializedEvent 事件



### 三、ApplicationListener 实现类

#### 1、ConfigFileApplicationListener

a）ApplicationEnvironmentPreparedEvent 事件

​		实例化并排序 EnvironmentPostProcessor 的类，回调 postProcessEnvironment 方法。

#### 2、DelegatingApplicationListener

a）ApplicationEnvironmentPreparedEvent 事件

​		实例化并排序 context.listener.classes 变量指定的类，放入广播器。

b）后续事件

​		向上一步加载的实例广播事件。

#### 3、ConfigFileApplicationListener

​		这个类会去加载 spring 的配置文件到 environment 里面



### 四、重点区域

#### 1、加载 bean 定义

##### ConfigurationClassPostProcessor -----> BeanDefinitionRegistryPostProcessor

​		这个类首先处理预加载的 beanDefinition 数据，解析其中带 Configuration 注解的，这是主类上必须有这个注解的原因。

​		然后通过 ConfigurationClassParser.doProcessConfigurationClass 方法去解析这个 Configuration 类。这个方法内部依次去加载 Component 的内部类，处理 PropertySource 属性，处理 ComponentScan 注解（这是 SpringBoot 可以自动扫描 Component 的原因），处理 Import 注解引入的类，处理 ImportResource 注解引入的资源，把 Bean 注解的方法放入集合以便后续处理。

​		处理 ComponentScan 时，会通过 ComponentScanAnnotationParser 解析配置的各个属性，如果没有 basePackages，先找 basePackageClasses 的 package，如果没有则以当前 Configuration 类的 package 作为 basePackages。扫描这些包下面的所有资源，过滤出符合条件的（Component、Configuration、Optional条件），然后递归处理新扫描到的 Configuration 的类。

#### 2、初始化单例 bean

##### DefaultListableBeanFactory  ——>  AbstractAutowireCapableBeanFactory  ——>  DefaultSingletonBeanRegistry  ——>  AbstractBeanFactory

​		DefaultSingletonBeanRegistry 这个类保存了所有的 bean 的实例。通过三级缓存来解决循环依赖和代理对象问题。一级缓存 singletonObjects 保存了所有已经初始化完成的 bean，二级缓存 earlySingletonObjects 保存了已经实例化但还没有完全初始化的 bean，三级缓存保存用于产生 bean 的工厂。查找二级缓存的条件是当前 bean 正在创建，查找三级缓存可以通过参数控制。

​		先查找缓存，有的话直接返回（新对象肯定没有，循环依赖的话会找到三级缓存，然后把值放到二级缓存）。

​		拿到构造方法，通过反射创建对象，在初始化之前放一个工厂到三级缓存（这个工厂处理 **SmartInstantiationAwareBeanPostProcessor** 接口，AOP 会在这一步产生代理对象而替换原对象，导致最终的引用改变）。

​		然后填充对象（Autowired等），这一步可能产生循环依赖。

​		初始化 bean，BeanFactory 接口注释中列举的 bean 的生命周期在这一步完成。AOP 通过内部缓存来得到三级缓存工厂是否完成代理的信息，并在 **BeanPostProcessor**.postProcessAfterInitialization 进行代理。他们两步是配合并且只会执行一次。

​		最后再尝试从前两级缓存中去拿这个 bean，这是为了在循环依赖的情况下拿到和 B 对象一样的 A 的引用。



#### 3、scope 的实现

##### RequestContextListener、AbstractRequestAttributesScope、RequestContextHolder

​		RequestContextHolder 通过 ThreadLocal 保存和当前线程相关的数据。

​		AbstractRequestAttributesScope.get 方法会去 RequestContextHolder 取当前线程的数据。

​		RequestContextListener 在每个请求开始和结束之前，会去设置 RequestContextHolder 内的值。



#### 4、bean 的生命周期

​		在 BeanFactory 的注释上有详细的描述。

​		代码内写死的：BeanNameAware ——> BeanClassLoaderAware ——> BeanFactoryAware

​		ApplicationContextAwareProcessor：EnvironmentAware ——> EmbeddedValueResolverAware ——> ResourceLoaderAware ——> ApplicationEventPublisherAware ——> MessageSourceAware ——> ApplicationContextAware

​		WebApplicationContextServletContextAwareProcessor：ServletContextAware ——> ServletConfigAware

​		BeanPostProcessor.postProcessBeforeInitialization ——> InitializingBean ——> init method ——> postProcessAfterInitialization

​		销毁：DestructionAwareBeanPostProcessor ——> DisposableBean ——> destroy-method



#### 5、ImportBeanDefinitionRegistrar



### 五、最重要的接口

#### 1、BeanFactory

​		bean 工厂，职责包括创建 bean 的实例，保存所有单例 bean。

#### 2、ApplicationContext

​		应用上下文，本身实现了 BeanFactory 接口，但是 AbstractRefreshableApplicationContext 内部保存了一个 DefaultListableBeanFactory 的实例，所有 BeanFactory 方法都调用这个实例的方法去执行。

#### 3、AliasRegistry ——> BeanDefinitionRegistry

​		可以注册别名和 bean 定义

#### 4、BeanPostProcessor ——> InstantiationAwareBeanPostProcessor ——> SmartInstantiationAwareBeanPostProcessor

​		这几个接口会在 bean 实例化前后调用，允许控制实例化的方式，也可以对实例化之后的 bean 进行修改。

#### 5、BeanFactoryPostProcessor ——> BeanDefinitionRegistryPostProcessor

​		这几个接口在 BeanFactory 和 BeanDefinitionRegistry 实例化之后被调用，允许对 BeanFactory 和 BeanDefinitionRegistry 进行自定义的操作。

#### 6、ApplicationContextInitializer

​		在 context 初始化之前对 context 进行定制。

#### 7、ApplicationListener

​		监听启动过程中和运行时的各种事件。

#### 8、Environment

​		环境变量

#### 9、Aware

​		在 bean 初始化的不同阶段，会回调相应的子接口。



### 六、最重要的类

#### 1、AnnotationConfigServletWebServerApplicationContext

​		web 类型的 SpringBoot 实际使用的 ApplicationContext 类

#### 2、DefaultListableBeanFactory

​		实际使用的 BeanFactory 的类。



### 七、AOP

#### 1、AbstractAutoProxyCreator

​		初始化 bean 的时候，发现有 AnnotationAwareAspectJAutoProxyCreator 这个类定义，则回调到父类的 postProcessAfterInitialization 方法，如果发现是需要进行 AOP 的类，则会创建一个动态代理。



### 八、杂

#### 1、自动配置

​		