---
title: Spring Boot：创建Spring Boot应用
date: 2024-04-01 11:20:56 +0800
categories: [编程, Spring Boot]
tags:
- Spring Boot
- parent
---

## 版本

Spring当年出现时，为解决各种Jar包版本兼容性问题作出了突出的贡献。但今时今日，Spring迭代了十余年，其自身的版本兼容问题也到了不得不重视的时候。

截止到今天（2024年4月1日），Spring最新版本已经来到了`3.2.4`。但需要注意的是，自`3.0.13`版本开始，Spring Boot所依赖的最低JDK版本是**Java 17**。如果你希望使用**Java 8**进行开发，那么你能使用Spring Boot的最高版本是`2.7.18`，该版本已在2023年11月23日停止更新。

你可以在这里看到Spring Boot的版本支持情况：https://spring.io/projects/spring-boot#support

你也可以在这里找到Spring Boot各版本文档：https://spring.io/projects/spring-boot#learn

鉴于当前使用最多的Java版本依然是**Java 8**，这里还是选择Spring Boot的`2.7.18`版本。

Spring Boot 2.7.18版本文档：https://docs.spring.io/spring-boot/docs/2.7.18/reference/htmlsingle/

## 引入Spring Boot父工程

Spring Boot的使用非常简单、方便，创建一个普通的Maven工程，编辑`pom.xml`如下即可完成Spring Boot的引入：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>top.szhome</groupId>
    <artifactId>szhome-admin</artifactId>
    <version>1.0.0-SNAPSHOT</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.18</version>
    </parent>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
	
    // 其他部分省略...
</project>
```

如果你使用IDEA开发工具，按下`command`键的同时，鼠标点击`<artifactId>spring-boot-starter-parent</artifactId>`行，你会进入到`spring-boot-starter-parent`的pom描述，同样的操作可以进入到`spring-boot-starter-parent`的父工程`spring-boot-dependencies`，这里罗列了`Spring Boot 2.7.18`版本所依赖的各种Jar包的版本，并为你的工程管理这些Jar包版本。

上面指定工程的父工程为`spring-boot-starter-parent`并不会为你引入任何Jar包，它的作用只是帮你管理Spring Boot涉及的Jar包版本，避免一些版本不兼容的问题。

## 开发web应用

为了简化我们的开发难度，Spring Boot提供了一系列的名为`Starter`的Jar包，这些Jar包就像开关一样，你只要引入了某个`Starter`，就打开了Spring Boot为你提供的对应`Starter`的功能。比如：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

像上面一样引入一个`spring-boot-starter-web`的Jar包依赖，web开发相关的依赖就会被引入进来，包括`Servlet`、`Spring MVC`、`Jackson`、日志等等。你无需考虑要如何配置`Servelt`到`URL`的映射，也不用再配置`web.xml`等，你可以直接编写你的 接口代码。像下面这样：

```java
@SpringBootApplication
@RestController
public class SzhomeAdminApplication {

    public static void main(String[] args) {
        SpringApplication.run(SzhomeAdminApplication.class, args);
    }

    @GetMapping
    public String hello() {
        return "hello, world";
    }
}
```

- `main`方法：main方法启动了一个`SpringApplication`实例，指定了主要的启动配置类为当前类，并将启动脚本入参透传；
- `@SpringBootApplication`注解：该注解是一个组合注解，它组合了以下三个注解：
  - `@SpringBootConfiguration`注解，这是一个特殊的`@Configuration`注解，标记类是一个Spring Boot应用配置类；
  - `@EnableAutoConfiguration`注解，标记应用启用自动配置功能；
  - `@ComponentScan`注解，标记应用启动时，扫描自动注册Bean的范围。
- `@RestController`注解，标记当前类是一个`Controller`类；
- `@GetMapping`注解，声明该方法对应一个接口处理器。

执行`main`方法，待应用启动完成，访问`http://localhost:8080`，即可访问`hello`方法对应的接口，看到`hello, world`的响应结果。

## 打包发布

当我们创建一个Java工程后，我们通常可以使用这样的命令打包应用：

```shell
% mvn clean package

[INFO] Scanning for projects...
[INFO] 
[INFO] ----------------------< top.szhome:szhome-admin >-----------------------
[INFO] Building szhome-admin 1.0.0-SNAPSHOT
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- clean:3.2.0:clean (default-clean) @ szhome-admin ---
[INFO] Deleting /szhome-admin/target
[INFO] 
[INFO] --- resources:3.2.0:resources (default-resources) @ szhome-admin ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Using 'UTF-8' encoding to copy filtered properties files.
[INFO] Copying 1 resource
[INFO] Copying 0 resource
[INFO] 
[INFO] --- compiler:3.10.1:compile (default-compile) @ szhome-admin ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /szhome-admin/target/classes
[INFO] 
[INFO] --- resources:3.2.0:testResources (default-testResources) @ szhome-admin ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Using 'UTF-8' encoding to copy filtered properties files.
[INFO] skip non existing resourceDirectory /szhome-admin/src/test/resources
[INFO] 
[INFO] --- compiler:3.10.1:testCompile (default-testCompile) @ szhome-admin ---
[INFO] Changes detected - recompiling the module!
[INFO] 
[INFO] --- surefire:2.22.2:test (default-test) @ szhome-admin ---
[INFO] 
[INFO] --- jar:3.2.2:jar (default-jar) @ szhome-admin ---
[INFO] Building jar: /szhome-admin/target/szhome-admin-1.0.0-SNAPSHOT.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  5.067 s
[INFO] Finished at: 2024-04-01T16:04:33+08:00
[INFO] ------------------------------------------------------------------------

```

如上，在工程下的**target**目录下就生产了Jar包文件，然后我们我们尝试启动应用：

```shell
 % java -jar ./target/szhome-admin-1.0.0-SNAPSHOT.jar
```

但很快就会发现行不通，我们将得到下面这样的报错：

```shell
./target/szhome-admin-1.0.0-SNAPSHOT.jar中没有主清单属性
```

不要慌，这是可以解决的，我们只需要在工程的`pom.xml`文件中加上如下配置，使用Spring Boot的Maven打包插件即可：

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

至于原因，你可以在这里找到：https://docs.spring.io/spring-boot/docs/2.7.18/reference/htmlsingle/#getting-started.first-application.executable-jar

简单来说，Java本身没有提供标准的嵌套Jar包（一个Jar包中包含多个其他Jar包）加载机制，这会导致我们在打包时遇到一些问题。一种比较常见的方式是将应用依赖的类文件全部打包到单个Jar包，但是这么做会导致你无法通过Jar包了解你的应用依赖了哪些Jar，同时也无法解决多个Jar包存在同名类的问题。

因此Spring Boot采取了另一种方案来解决嵌套Jar包的问题，因此这里需要引入`spring-boot-maven-plugin`插件。

