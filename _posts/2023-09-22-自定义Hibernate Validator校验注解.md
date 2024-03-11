---
title: "自定义Hibernate Validator校验注解"
date: "2023-09-22"
categories: [编程, "Spring Boot"]
---

## 自定义注解

- @Constraint指明校验逻辑实现类；
- 至少应包含message、groups、payload三个属性；
  - message，校验不通过时的提示消息
  - groups，校验分组，指定校验生效的分组
- message支持属性文件ValidationMessages.properties；
- @Target可以是FIELD，也可以是TYPE等等

```java
/**
 * 名称校验
 *
 * @author 唐加利
 * @date 2023/9/22
 */
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = NameValidator.class)
public @interface Name {

    /**
     * 校验未通过的提示消息
     *
     * @return
     */
    String message() default "{szhome.validator.Name.message}";

    /**
     * 校验分组
     * @return
     */
    Class<?>[] group() default {};

    /**
     * 荷载信息
     * @return
     */
    Class<? extends Payload>[] payload() default {};

}
```

## 编写校验逻辑

### JDK版本



### Hibernate Validator版本

