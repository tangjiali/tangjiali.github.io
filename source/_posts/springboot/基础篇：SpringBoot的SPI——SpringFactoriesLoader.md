---
title: 基础篇：SpringBoot的SPI——SpringFactoriesLoader
date: 2021-01-31 22:18:00
toc: true
categories:
    - SpringBoot源码阅读
        - 基础篇
tag:
    - SpringBoot
    - 源码阅读
    - SpringFactoriesLoader
    - spring.factories
    - SPI
thumbnail: /imgs/illustration/2020-01-28-spring-factories-loader-and-spring-factories.svg
---

# SpringFactoriesLoader类注释

SpringBoot对`SpringFactoriesLoader`的定位是框架内部通用的工厂加载机制。

`SpringFactoriesLoader`会从`META-INF/spring.factories`文件中加载并实例化给定类型的工厂，而这些文件可能存在于多个类路径（`classpath`）下的多个Jar包中。

<!-- more -->

`spring.factories`文件的内容必须是属性文件（`Properties`）格式，同时`key`必须完整的接口名或抽象类名，`value`必须是这些接口或抽象类的实现类或子类名，多个实现类或子类以半角逗号（`,`）分割。格式如下：

```properties
example.MyService=example.MyServiceImpl1,example.MyServiceImpl2
```

这里的`example.MyService`是接口名，而`example.MyServiceImpl1`和`example.MyServiceImpl2`这个接口的两个实现类。

# SpringFactoriesLoader定义

`SpringFactoriesLoader`类除包含一个静态常量字符串`FACTORIES_RESOURCE_LOCATION`以外，另有2个成员变量、1个私有构造方法、2个共有方法和2个私有方法，定义如下图所示：

{% plantuml %}
    class SpringFactoriesLoader {
        - {static} logger: Log
        - {static} cache: Map<ClassLoader, MultiValueMap<String, String>>
        
        - {static} SpringFactoriesLoader()
        + {static} loadFactories(Class<T> factoryType, @Nullable ClassLoader classLoader) : List<T>
        + {static} loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) :  List<String>
        - {static} loadSpringFactories(@Nullable ClassLoader classLoader) : Map<String, List<String>>
        - {static} instantiateFactory(String factoryImplementationName, Class<T> factoryType, ClassLoader classLoader) : T
    }
{% endplantuml %}

## 构造方法-SpringFactoriesLoader

`SpringFactoriesLoader`的定位是一个工具类，不需要实例化。为此，SpringBoot做了两件事来防止实例化`SpringFactoriesLoader`类：

1. 构造方法私有化，`SpringFactoriesLoader`类唯一的构造方法使用类`private`修饰符，这意味着外部是无法直接通过构造方法创建它的实例，也不能通过反射来创建实例。
2. final修饰符，`SpringFactoriesLoader`类本身使用了`final`修饰符，因此它是不能被其他类继承，也就不会通过创建子类实例来使用它的能力。

## 实例化工厂-instantiateFactory

`instanticateFactory`方法签名如下：
```java
private static <T> T instantiateFactory(String factoryImplementationName, Class<T> factoryType, ClassLoader classLoader)
```

这个方法有三个入参，分别是：

+ `String factoryImplementationName`，工厂实现类名称，即要实例化的工厂类；
+ `Class<T> factoryType`，工厂类型，一个接口或抽象类；
+ `ClassLoader classLoader`，加载要实例化的类的类加载器。

看这个方法的实现：

```java
@SuppressWarnings("unchecked")
private static <T> T instantiateFactory(String factoryImplementationName, Class<T> factoryType, ClassLoader classLoader) {
    try {
        Class<?> factoryImplementationClass = ClassUtils.forName(factoryImplementationName, classLoader);
        if (!factoryType.isAssignableFrom(factoryImplementationClass)) {
            throw new IllegalArgumentException(
                    "Class [" + factoryImplementationName + "] is not assignable to factory type [" + factoryType.getName() + "]");
        }
        return (T) ReflectionUtils.accessibleConstructor(factoryImplementationClass).newInstance();
    }
    catch (Throwable ex) {
        throw new IllegalArgumentException(
            "Unable to instantiate factory class [" + factoryImplementationName + "] for factory type [" + factoryType.getName() + "]",
            ex);
    }
}
```

首先是使用指定的类加载器加载`factoryImplementationName`指定的类，如果加载失败，则抛出异常，此时在`catch`代码块中被处理成`IllegalArgumentException`异常。

加载到指定工厂实现类后，接着检测该类是否为指定工厂类型的实现或子类，如果不是，同样抛出`IllegalArgumentException`异常。

最后通过反射调用工厂实现类的无参构造方法生成工厂实现类的实例，并返回。

简单来说，这个私有方法是个纯粹的工具方法。它通过指定类名，限定类名实现的接口，反射生成指定类名类的实例。如果指定的类名不存在对应的类，或者指定类名对应的类不是指定的接口的实现类，都将抛出非法参数异常。

## 加载Spring工厂-loadSpringFactories

`loadSpringFactories`方法签名如下，它只有一个可空的入参，类型为类加载器：

```java
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader)
```

它的作用加载入参类加载器加载的那些jar包中`META-INF/spring.factories`文件，并将结果组织为接口到实现类到映射。方法实现如下：

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
        throw new IllegalArgumentException("Unable to load factories from location [" +
                FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```

从源码可以看出，这个方法用到了缓存。它先是尝试从缓存中获取结果，如果已经存在想要的结果，就直接返回。

当缓存中尚没有指定类加载器加载的Spring工厂时，真正获取Spring工厂的代码再来处理加载逻辑。可以看到，如`SpringFactoriesLoader`的类注释中所描述的，这个方法会加载所有的`META/spring.factories`文件，当然，任意一个jar包中都可能包含这样一个文件，因此Spring使用了可枚举类型保存这些`META/spring.factories`文件。

接下来就是按属性文件逐个处理这些文件，属性文件由key-value组成，这些文件也是key-value形式。每一个key都是一个接口或抽象类的完整类名，key后面对应的value则是key对应接口或抽象类的实现类或子类，多个实现类或子类以逗号分割。这个方法将相同的key作为字典key，value分割后以`List`的形式作为字典value。当然，处于不同jar包中的`spring.factories`文件如果存在相同的key，它们对应的value也会被聚合到同一个key对应的字典value。

最终，所有的`spring.factories`文件都将被处理为接口名为key，实现类为value的字典，并将这个字典返回。同时，`loadSpringFactories`方法也会将这个字典作为缓存数据，入参的类加载器作为缓存key，保存到缓存中。

## 加载工厂名称-loadFactoryNames

`loadFactoryNames`方法用于从`META-INF/spring.factories`文件加载指定工厂类型的工厂名称集合，实现如下：

```java
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
	String factoryTypeName = factoryType.getName();
    return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
}
```

已知`loadSpringFactories`方法已经将所有jar包中的`META-INF/spring.factories`文件处理为以接口或抽象类为key的字典集合，并放入缓存，这里很容易得出，某个指定接口或抽象类对应的实现类或子类类名集合。


## 加载工厂-loadFactories

`loadFactories`方法是`SpringFactoriesLoader`类作用的体现者，它集成了前述几个方法的能力，实现从`META-INF/spring.factories`中加载、实例化工厂。下面是它的实现：

```java
public static <T> List<T> loadFactories(Class<T> factoryType, @Nullable ClassLoader classLoader) {
    Assert.notNull(factoryType, "'factoryType' must not be null");
    ClassLoader classLoaderToUse = classLoader;
    if (classLoaderToUse == null) {
        classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
    }
    List<String> factoryImplementationNames = loadFactoryNames(factoryType, classLoaderToUse);
    if (logger.isTraceEnabled()) {
        logger.trace("Loaded [" + factoryType.getName() + "] names: " + factoryImplementationNames);
    }
    List<T> result = new ArrayList<>(factoryImplementationNames.size());
    for (String factoryImplementationName : factoryImplementationNames) {
        result.add(instantiateFactory(factoryImplementationName, factoryType, classLoaderToUse));
    }
    AnnotationAwareOrderComparator.sort(result);
    return result;
}
```

先是入参`factoryType`不能为空的校验，接着是在入参类加载器为空时使用默认的加载器，再然后是通过`loadFactoryNames`方法的能力获得指定类型`factoryType`在`spring.factories`文件中对应的实现类集合。

拿到实现类集合后，再借助`instantiateFactory`方法逐个实例化这些工厂实现类，并加入到结果集。

最后，再返回结果集之前，对结果集做了排序处理。

# SpringFactoriesLoader与SPI

## SPI

在Java中有一个名为`ServiceLoader`的类，与`SpringFactoriesLoader`有着相近的能力，那就是从文件中加载并实例化接口实现类。不同是`ServiceLoader`类是从`META-INF/services`目录中加载文件，且文件名须是接口完整类名称，文件内容则是接口实现类的完整名称。

实际上这两种实现都是一种服务发现机制，即**service provider interface**，简称**SPI**。

## 使用JDK提供的SPI

JDK本身提供的SPI能力使用步骤如下：
1. 在类路径下创建`META-INF/services`目录；
2. 以接口完整名称为文件名，在`META-INF/services`目录下创建文件；
3. 将该接口实现类逐行写入这个文件中。

接下来就是编码：

```java
// 获取指定接口的全部实现
ServiceLoad<MyInterface> myIntfs = ServiceLoader.load(MyInterface.class);

// 调用实现类方法
for(MyInterface myIntf : myIntfs) {
    myIntf.doSomething();
}
```

# 附录

## spring-boot.jar下的spring.factories

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