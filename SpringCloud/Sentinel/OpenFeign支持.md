---
title: OpenFeign整合Sentinel
categories:
- [SpringCloud, Sentinel]
tags:
- SpringCloud
- Sentinel
- OpenFeign
---



## OpenFeign支持

- 配置文件打开 sentinel 对 feign 的支持：`feign.sentinel.enabled=true`
- 加入 `openfeign starter` 依赖使 `sentinel starter` 中的自动化配置类生效：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

> Feign 对应的接口中的资源名策略定义：httpmethod:protocol://requesturl。`@FeignClient` 注解中的所有属性，Sentinel 都做了兼容。

```java
@FeignClient(contextId = "RemoteFanyiService", name = "demo-fanyi", fallbackFactory = RemoteFanyiFallbackFactory.class)
public interface RemoteFanyiService {
  @GetMapping("/api/baidu/translate")
  String translate(@RequestParam(value = "query") String query,
                   @RequestParam(value = "from") String from,
                   @RequestParam(value = "to") String to);
}
```

`RemoteFanyiService` 接口中方法 `translate` 对应的资源名为 `GET:http://demo-fanyi/api/baidu/translate`。

资源名策略定义部分源码：

com.alibaba.cloud.sentinel.feign.SentinelInvocationHandler 

```java
 String resourceName = methodMetadata.template().method().toUpperCase() + ":" + hardCodedTarget.url() + methodMetadata.template().path();
```

## 参考

Spring Cloud Alibaba Sentinel Wiki文档：https://github.com/alibaba/spring-cloud-alibaba/wiki/Sentinel
