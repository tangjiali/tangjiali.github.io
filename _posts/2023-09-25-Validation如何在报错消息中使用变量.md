---
title: "Validation如何在报错消息中使用变量"
date: "2023-09-25"
categories: [编程, "Spring Boot"]
tags:
- Hibernate Validator
- 参数校验
- 重构
---

你是不是经常这么写参数校验：

```java
@Data
public class User {
  
  /**
   * 用户名
   */
  @Length(min = 3, max = 64, message = "用户名长度应在3到64个字符之间")
  private String name;
  
}
```

有一天业务调整，用户名的长度范围改为了5～32，你又要这么写：

```java
@Data
public class User {
  
  /**
   * 用户名
   */
  @Length(min = 5, max = 32, message = "用户名长度应在5到32个字符之间")
  private String name;
  
}
```

这样不是不能完成需求，但是本着`Don't Repeat Yourself`的原则，写两次最大最小长度、改两次最大最小长度，确实不太优雅。

## 默认消息插入器

