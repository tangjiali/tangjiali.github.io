---
title: "使用OpenFeign实现远程调用"
date: "2023-09-22"
categories: [编程, "Spring Cloud"]
---

## 注册Feign客户端

```java
@FeignClient(name = "szhome-system", contextId= "user", fallback = UserClientFallback.class)
public interface UserClient {
  
  /**
   * 根据ID获取用户信息
   * @param id 用户ID
   * @return 返回用户信息
   */
  @GetMapping("/user/{id}")
  @Validate
  User getUser(@PathVariable @Valid @NotNull(message = "用户ID不能为空") Long id);
  
}
```

如上代码，通过`@FeignClient`注解定义了一个Feign客户端。

