---
title: 基础篇：SpringBoot的事件发布机制——SpringApplicationRunListener
date: 2021-02-01 12:00:00
toc: true
categories:
    - SpringBoot源码阅读
        - 基础篇
tag:
    - SpringBoot
    - 源码阅读
    - SpringApplicationRunListener
    - spring.factories
    - SPI
thumbnail: /imgs/illustration/基础篇：SpringBoot的事件发布机制——SpringApplicationRunListener.jpeg
---

# SpringApplicationRunListener

## 类注释

`SpringApplicationRunListener`是一个接口，用于监听`SpringApplication`类下`run`方法的执行，且每次执行`run`方法都应该创建一个新的`SpringApplicationRunListener`实例去监听。

`SpringApplicationRunListener`类的实现类是通过`SpringFactoriesLoader`加载的，且要求该类的实现类应当有一个入参为`SpringApplication`类型和`String[]`类型的共有构造方法。

`SpringApplicationRunListener`接口定义如下：

{% plantuml %}
interface SpringApplicationRunListener {
    + starting(): void
    + environmentPrepared(ConfigurableEnvironment environment): void
    + contextPrepared(ConfigurableApplicationContext context): void
    + contextLoaded(ConfigurableApplicationContext context): void
    + started(ConfigurableApplicationContext context): void
    + running(ConfigurableApplicationContext context): void
    + failed(ConfigurableApplicationContext context, Throwable exception): void
}
{% endplantuml %}

<!-- more -->

## 实现类

目前SpringBoot中`SpringApplicationRunListener`接口只有一个实现类，即`EventPublishingRunListener`，结构如下：

{% plantuml %}
class EventPublishingRunListener implements SpringApplicationRunListener {
    - SpringApplication application
    - String[] args
    - SimpleApplicationEventMulticaster initialMulticaster

    + EventPublishingRunListener(SpringApplication application, String[] args)

}
{% endplantuml %}

如上，`EventPublishingRunListener`持有`SpringApplication`实例，同时还持有一个事件多播器`SimpleApplicationEventMulticaster`。当`EventPublishingRunListener`监听器被实例化时，如`SpringApplicationRunListener`约定，调用了如下构造方法：

```java
public EventPublishingRunListener(SpringApplication application, String[] args) {
    this.application = application;
    this.args = args;
    this.initialMulticaster = new SimpleApplicationEventMulticaster();
    for (ApplicationListener<?> listener : application.getListeners()) {
        this.initialMulticaster.addApplicationListener(listener);
    }
}
```

可见，事件多播器将`SpringApplication`的监听器集合加入到了自己的监听者队列。

# 事件发布与监听

## 事件发布

`EventPublishingRunListener`作为`SpringApplicationRunListener`的唯一实现，承担了`SpringApplication`调用`run`方法执行过程中的所有事件发布任务。

`EventPublishingRunListener`的每一个方法都会发布一个事件，事件发布要么由事件多播器完成，如：

```java
@Override
public void starting() {
    this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
}
```

要么由应用上下文完成，如：

```java
@Override
public void started(ConfigurableApplicationContext context) {
    context.publishEvent(new ApplicationStartedEvent(this.application, this.args, context));
}
```

`EventPublishingRunListener`发布的事件类型有如下几种：

+ `ApplicationStartingEvent`，应用启动事件
+ `ApplicationEnvironmentPreparedEvent`，应用环境就绪事件
+ `ApplicationContextInitializedEvent`，应用上下文初始化完成事件
+ `ApplicationPreparedEvent`，应用上下文就绪事件
+ `ApplicationStartedEvent`，应用已启动事件
+ `ApplicationReadyEvent`，应用就绪事件
+ `ApplicationFailedEvent`，应用启动失败事件

## 事件多播器

`EventPublishingRunListener`使用`SimpleApplicationEventMulticaster`作为事件多播器，用于发布事件，并通知监听者处理事件。

{% plantuml %}
class AbstractApplicationEventMulticaster {
    - ListenerRetriever defaultRetriever
    - Map<ListenerCacheKey, ListenerRetriever> retrieverCache
    - ClassLoader beanClassLoader
    - ConfigurableBeanFactory beanFactory
    - Object retrievalMutex
}

class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster{
    
	- Executor taskExecutor

	- ErrorHandler errorHandler

    + multicastEvent(ApplicationEvent event): void
    + multicastEvent(final ApplicationEvent event, ResolvableType eventType): void
}
{% endplantuml %}






<div style="display:none">

# 事件发布

`SpringApplicationRunListener`是监听器，监听的是`SpringApplication`类`run`方法的执行。这个`run`方法不是一下子就执行完的，它分几个阶段，而`SpringApplicationRunListener`监听每个阶段体现在它的多个方法上。

## starting——开始执行事件

`starting`方法在`run`方法第一次执行时会被立即调用，通常用于做一些前期初始化的工作。`EventPublishingRunListener`中实现如下：

```java
@Override
public void starting() {
    this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
}
```

如上，`EventPublishingRunListener`通过多播器广播了一条事件消息，事件类型为`ApplicationStartingEvent`，事件源是`this.application`，携带数据`this.args`。监听者监听到该消息后，应当知道此时`SpringApplication`调用了`run`方法，且环境变量、应用上下文都未准备就绪，不可使用。

## environmentPrepared——环境就绪事件

`environmentPrepared`方法在环境`Environment`准备就绪的时候执行，实现如下：

```java
@Override
public void environmentPrepared(ConfigurableEnvironment environment) {
    this.initialMulticaster
            .multicastEvent(new ApplicationEnvironmentPreparedEvent(this.application, this.args, environment));
}
```
同样是通过多播器广播了一条事件消息，事件类型为`ApplicationEnvironmentPreparedEvent`，事件源是`this.application`，携带数据`this.args`，已经准备就绪的`Environment`也在事件中带入。此时监听者应当知道应用已经准备好了`Environment`数据，可以直接使用。但此时应用上下文还没有被创建，不可使用。

## contextPrepared——应用上下文就绪事件

`contextPrepared`方法执行时，意味着`ApplicationContext`已经被创建，且准备就绪。但此时应用上下文并没有真正加载sources。

```java
@Override
public void contextPrepared(ConfigurableApplicationContext context) {
    this.initialMulticaster.multicastEvent(new ApplicationContextInitializedEvent(this.application, this.args, context));
}
```

如上，这里的事件类型为`ApplicationContextInitializedEvent`。

## contextLoaded——应用上下文加载完毕事件

`contextLoaded`

## started——应用启动事件

## running——应用运行中事件

## failed——启动失败事件

</div>