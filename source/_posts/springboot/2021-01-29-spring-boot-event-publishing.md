---
title: SpringBoot的事件发布机制
date: 2021-01-29
tag:
    - SpringBoot
    - Java
    - 事件发布机制
    - SpringApplicationRunListener
categories:
    - SpringBoot源码分析
---

# SpringApplicationRunListener

SpringApplicationRunListener 是一个接口，也是一个监听器，用于监听 SpringApplication.run 方法的执行过程。

<!-- more -->

《SpringFactoriesLoader与spring.factories》中分析了SpringBoot启动时是如何加载  SpringApplicationRunListeners 类的实例的，结合 spring-boot.jar 包中的 META-INF/spring.factories 文件配置内容，可知该类的成员变量 listeners 只有一个元素，且实现类是 EventPublishingRunListener ：

```properties
# Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener
```