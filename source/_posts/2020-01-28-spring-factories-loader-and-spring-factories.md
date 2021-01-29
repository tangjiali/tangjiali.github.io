---
title: SpringFactoriesLoader与spring.factories
tag: 
    - SpringBoot
    - Java
    - spring.factories
    - SpringApplicationRunListener
categories: 
    - SpringBoot源码分析
---

# 获取SpringApplicationRunListeners
当通过 SpringApplication.run 方法启动一个SpringBoot应用时，靠前位置即有如下代码：

```java
SpringApplicationRunListeners listeners = getRunListeners(args);
```

显然，这里定义来一个SpringApplicationRunListeners类型的变量，并调用了 getRunListener 方法获取其实例对象。

<!-- more -->

查看源码可知， SpringApplicationRunListeners 是 SpringApplicationRunLinstener 的容器。结构如下所示，它有一个带参构造方法，入参分别是 Log 和 SpringApplicationRunLinstener集合。

![avatar](/imgs/illustration/2020-01-28-spring-factories-loader-and-spring-factories.svg)

再来看 getRunListeners 方法是如何拿到一个 SpringApplicationRunListeners 的实例的。如下，在 getRunListeners 方法中，首先构造了一个Class类的数组，接着调用了 SpringApplicationRunListeners 的带参构造方法。其中，第一个参数复用了 SpringApplication 的 logger 成员变量，第二个参数则通过 getSpringFactoriesInstances 方法获取监听器集合。

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
    Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
    return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```

如此，即构造出了SpringApplicationRunListeners 类的实例对象。

# 创建Spring工厂实例

如前所述，SpringApplicationRunListeners 对象虽然构造出来了，但该类构造方法的第二个参数 listeners ，我们只知道是通过 getSpringFactoriesInstances 方法获取的，至于具体是如何拿到的尚未清楚。

继续跟踪 getSpringFactoriesInstances 方法，该方法定义如下：

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

显然，该方法有三个入参，分别是 Class<T> type 、 Class<?>[] 以及可变参数 Object... args ；出参 Collection<T> 是一个集合；范型参数 T 取决于第一个入参 type 。

通过前面的分析可知，当调用该方法获取 SpringApplicationRunLinstener集合时，入参分别如下：

+ type， SpringApplicationRunListener ；
+ parameterTypes， new Class<?>[] { SpringApplication.class, String[].class } ；
+ args， new Object[]{this, args} ，其中 this 表示 SpringApplication 的实例， args 是 main 方法入参。

逐行分析 getSpringFactoriesInstances 方法的方法体：

1. 第一行是获取一个类加载器 classLoader ，源码比较简单，要么是从启动 SpringApplication 时指定的 resourceLoader 中获取，要么使用默认的类加载器； 
2. 定义一个字符串集合 names ，其元素通过 SpringFactoriesLoader.loadFactoryNames 方法加载而来；
3. 使用 getSpringFactoriesInstances 方法入参和 names 集合，创建出一个实例集合；
4. 对第3步创建的实例集合排序；
5. 返回排序后的实例集合。

跳过第2步，先来看当我们拿到 names 集合后，是如何创建实例的：

```java
private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes,
            ClassLoader classLoader, Object[] args, Set<String> names) {
    List<T> instances = new ArrayList<>(names.size());
    for (String name : names) {
        try {
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            Assert.isAssignable(type, instanceClass);
            Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
            T instance = (T) BeanUtils.instantiateClass(constructor, args);
            instances.add(instance);
        }
        catch (Throwable ex) {
            throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, ex);
        }
    }
    return instances;
}
```

createSpringFactoriesInstances 方法比较简单，就是遍历了入参 names ，对每一个name做如下处理得到：

1. 以name为类名，尝试加载该类；
2. 校验，加载到到类，须是入参 type 的子类；
3. 获取加载到的类的指定参数的构造方法；
4. 使用构造方法和入参，实例化加载的类；
5. 将实例加入到实例集合；

经过以上5步，得到实例集合后，将该实例集合返回。以上几步，无论哪一步发生异常，都会抛出异常，终止创建实例。

至此我们可以知道，当我们使用前面参数调用 getSpringFactoriesInstances 方法获取 SpringApplicationRunLinstener 接口的实例集合时，会调用该接口实现类的构造方法，且构造方法入参有两个：
1. 形参 SpringApplication ，实参为应用入口调用 SpringApplication.run 时内部创建的 SpringApplication 的实例；
2. 形参 String[] ，实参为 main 方法的入参，通常通过命令行指定。

回过头再来看 getSpringFactoriesInstances 方法，还剩下面这行加载 names 的代码没有分析：

```java
Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
SpringFactoriesLoader.loadFactoryNames
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
    String factoryTypeName = factoryType.getName();
    return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
}
```
如上， loadFactoryNames 方法有两个入参， Class<?> factoryType 和 ClassLoader classLoader ，其中 classLoader 可空。

loadFactoryNames 先是以 classLoader 为入参，调用类 loadSpringFactories 方法，得到一个 Map ；紧接着以 factoryType 类的类名为key，从这个 Map 中获取结果集。

再来看 loadSpringFactories 方法做了什么：

```java
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    MultiValueMap<String, String> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }
    try {
        Enumeration<URL> urls = (classLoader != null ?
                                 classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                                 ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        result = new LinkedMultiValueMap<>();
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();
                for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                    result.add(factoryTypeName, factoryImplementationName.trim());
                }
            }
        }
        cache.put(classLoader, result);
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```

从方法第一行代码可以看出，loadSpringFactories 方法使用缓存，保存缓存的 cache 定义如下：

```java
private static final Map<ClassLoader, MultiValueMap<String, String>> cache = new ConcurrentReferenceHashMap<>();
```

如果能从缓存中拿到结果，即直接返回缓存中的结果；但初次调用 loadSpringFactories 方法， cache 当然是空的。

接着看try块中的代码，首先是从classpath下加载 FACTORIES_RESOURCE_LOCATION 资源，即 META-INF/spring.factories 文件。由于每个jar包中都可能存在这样一个文件，因此这里是一个可枚举的对象 urls 。下面是 spring-boot-2.2.2-RELEASE.jar 包中的 META-INF/spring.factories 文件内容：

```properties
# PropertySource Loaders
org.springframework.boot.env.PropertySourceLoader=\
org.springframework.boot.env.PropertiesPropertySourceLoader,\
org.springframework.boot.env.YamlPropertySourceLoader
# Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener
# Error Reporters
org.springframework.boot.SpringBootExceptionReporter=\
org.springframework.boot.diagnostics.FailureAnalyzers
# Application Context Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
org.springframework.boot.context.ContextIdApplicationContextInitializer,\
org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
org.springframework.boot.rsocket.context.RSocketPortInfoApplicationContextInitializer,\
org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.ConfigFileApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.context.logging.ClasspathLoggingApplicationListener,\
org.springframework.boot.context.logging.LoggingApplicationListener,\
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener
# Environment Post Processors
org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor,\
org.springframework.boot.env.SpringApplicationJsonEnvironmentPostProcessor,\
org.springframework.boot.env.SystemEnvironmentPropertySourceEnvironmentPostProcessor,\
org.springframework.boot.reactor.DebugAgentEnvironmentPostProcessor
# Failure Analyzers
org.springframework.boot.diagnostics.FailureAnalyzer=\
org.springframework.boot.diagnostics.analyzer.BeanCurrentlyInCreationFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.BeanDefinitionOverrideFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.BeanNotOfRequiredTypeFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.BindFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.BindValidationFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.UnboundConfigurationPropertyFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.ConnectorStartFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.NoSuchMethodFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.NoUniqueBeanDefinitionFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.PortInUseFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.ValidationExceptionFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.InvalidConfigurationPropertyNameFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.InvalidConfigurationPropertyValueFailureAnalyzer
# FailureAnalysisReporters
org.springframework.boot.diagnostics.FailureAnalysisReporter=\
org.springframework.boot.diagnostics.LoggingFailureAnalysisReporter
```

loadSpringFactories 方法会遍历这个 urls 并逐个处理其中的元素，即每一个 META-INF/spring.factories 文件。可以看到，这个属性文件中，key是接口名，value为以逗号分割的接口实现类。最终结果是把所有META-INF/spring.factories 文件中的键值对，按接口名归并，一个接口对应一组实现类。

# 串联

最初，我们要构造一个 SpringApplicationRunListeners 的实例，如下：

```java
SpringApplicationRunListeners listeners = getRunListeners(args);
```

getRunListeners 方法中调用了带参构造方法，第一个参数复用了 SpringApplication 的 logger 成员变量，第二个参数则通过 getSpringFactoriesInstances 方法获取类型为 SpringApplicationRunListener 的实现类集合。

getSpringFactoriesInstances 首先收集散落在各个jar包中的 META-INF/spring.factories 文件，并将这些文件中的内容，按key做归并，完成接口到实现类的映射；接着轻松的拿到接口 SpringApplicationRunListener 所对应的实现类集合；然后再将这些实现类实例化，并返回。

至此，就获取到了 SpringApplicationRunListeners 的实例。

# 发散

显然我们也可以通过这种方式实现定制的业务逻辑，或者扩展SpringBoot。譬如，在项目中创建自己的 META-INF/spring.factories ，定义自己的 ApplicationListener ，使SpringBoot应用启动时可以做一些初始化的工作。