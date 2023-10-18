---
title: Spring Cloud Gateway基础知识点
categories:
- [SpringCloud, Gateway]
tags:
- SpringCloud
- Gateway
---



## 三个主要概念

1. **路由（Route）：** 路由是Spring Cloud Gateway的核心概念之一。路由定义了客户端请求的匹配规则和转发规则，用于将客户端的请求映射到后端的微服务上。路由规则由一个唯一的ID、一个目标URI（指向后端服务的地址）和一系列的Predicate（断言）和Filter（过滤器）组成。
2. **断言（Predicate）：** 断言用于匹配客户端请求的条件，如果请求匹配了断言的条件，则应用该路由规则，并将请求转发到指定的URI。断言可以匹配请求的路径、请求方法、请求头等信息。
3. **过滤器（Filter）：** 过滤器用于在请求被路由前后对请求和响应进行处理。过滤器可以修改请求和响应的内容，添加头信息，记录日志，进行权限校验等。Spring Cloud Gateway中有两种过滤器，分别是全局过滤器（Global Filter）和局部过滤器（GatewayFilter）。全局过滤器对所有的请求都生效，而局部过滤器只对特定的路由规则生效。

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          # 将服务ID转换为小写
          lowerCaseServiceId: true
          enabled: true
      # 路由。定义了客户端请求的匹配规则和转发规则
      routes: 
        - id: demo-fanyi
          # lb://实现基于负载均衡的服务路由
          uri: lb://demo-fanyi
          # 断言。定义路由的匹配条件，满足条件则转发到uri指定的服务
          predicates: 
            - Path=/fanyi/**
          # 过滤器。应用于路由的过滤器，用于修改请求或响应。
          filters: 
            - StripPrefix=1
```

## 工作流程

Spring Cloud Gateway的工作流程如下：

1. **客户端发送请求：** 客户端发送HTTP请求到Spring Cloud Gateway。
2. **Gateway Handler Mapping：** Gateway Handler Mapping将请求映射到对应的路由规则上。
3. **全局过滤器（Global Filter）：** 在请求进入Gateway Handler之前，先经过全局过滤器。全局过滤器对请求进行全局处理，如权限校验、日志记录等。全局过滤器可以对所有请求生效。
4. **路由规则匹配：** Spring Cloud Gateway根据路由规则（Route）将请求转发到对应的微服务实例上。路由规则可以通过配置文件或编程方式定义。
5. **请求转发：** 根据路由规则，Gateway将请求转发到后端的微服务。
6. **局部过滤器（Filter）：** 在请求转发到微服务前后，可以配置局部过滤器对请求和响应进行处理。局部过滤器只对特定的路由规则生效。
7. **微服务处理请求：** 请求到达后端微服务，微服务处理请求并生成响应。
8. **响应返回：** 微服务将处理完成的响应返回给Gateway。
9. **局部过滤器（Filter）处理响应：** 响应返回Gateway后，局部过滤器可以对响应进行处理，例如添加响应头、修改响应内容等。
10. **全局过滤器（Global Filter）处理响应：** 经过局部过滤器处理后，响应再次经过全局过滤器，全局过滤器可以对所有响应进行统一处理。
11. **响应返回客户端：** 经过过滤器处理后的响应最终返回给客户端。

## ServerWebExchange

`ServerWebExchange`是一个用于表示Web请求-响应交换的对象，在Spring WebFlux中用于处理HTTP请求。它包含了`ServerHttpRequest`和`ServerHttpResponse`对象，以及其他相关的上下文信息。

## HttpServletRequest和ServerHttpRequest

在`spring-boot-starter-webflux`中，使用的是Spring WebFlux框架，它采用了Reactive Streams标准，使用非阻塞的方式处理请求。在WebFlux中，不再依赖于`javax.servlet-api`，而是使用Netty或其他类似的Reactive Web服务器来处理请求。

如果项目是基于`spring-boot-starter-webflux`，并且需要使用`javax.servlet.http.HttpServletRequest`，可以通过注入`ServerHttpRequest`来获取与`HttpServletRequest`类似的功能。`ServerHttpRequest`是Spring WebFlux提供的Reactive方式获取HTTP请求信息的对象。

## mutate方法

`ServerWebExchange`对象本身是不可变的，无法直接修改其属性。因此，可以通过`mutate()`方法创建一个可变的`ServerWebExchange.Builder`对象，进行修改操作，并通过`build()`方法生成一个新的`ServerWebExchange`实例。

ServerHttpRequest 对象本身是不可变的，因此无法直接修改其属性，而是通过`mutate()`方法创建一个可变的请求构建器来进行修改。

网关鉴权全局过滤器中构建新的`ServerWebExchange`对象的示例如下：

```java
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain)
{
  // 获取当前线程的ServerHttpRequest
  ServerHttpRequest request = exchange.getRequest();
  // 原始请求对象本身是不可变的，因此无法直接修改其属性，使用mutate()方法创建一个可变的请求构建器
  ServerHttpRequest.Builder mutate = request.mutate();

  String url = request.getURI().getPath();
  // 跳过不需要验证的路径
  for (String pattern : ignoreWhite.getWhites()) {
    if (url.matches(pattern)) {
      return chain.filter(exchange);
    }
  }

  String token = getToken(request);
  if (StrUtil.isEmpty(token))
  {
    return unauthorizedResponse(exchange, "令牌不能为空");
  }
  Claims claims = JwtUtils.parseToken(token);
  if (claims == null)
  {
    return unauthorizedResponse(exchange, "令牌已过期或验证不正确！");
  }
  String userkey = JwtUtils.getUserKey(claims);
  boolean islogin = redisService.hasKey(getTokenKey(userkey));
  if (!islogin)
  {
    return unauthorizedResponse(exchange, "登录状态已过期");
  }
  String userid = JwtUtils.getUserId(claims);
  String username = JwtUtils.getUserName(claims);
  if (StrUtil.isEmpty(userid) || StrUtil.isEmpty(username))
  {
    return unauthorizedResponse(exchange, "令牌验证失败");
  }

  // 设置用户信息到请求
  addHeader(mutate, SecurityConstants.USER_KEY, userkey);
  addHeader(mutate, SecurityConstants.DETAILS_USER_ID, userid);
  addHeader(mutate, SecurityConstants.DETAILS_USERNAME, username);
  // 内部请求来源参数清除
  removeHeader(mutate, SecurityConstants.FROM_SOURCE);

  // 构建一个新的ServerHttpRequest对象
  ServerHttpRequest modifiedRequest = mutate.build();

  // ServerWebExchange本身是不可变的，无法直接修改其属性，使用mutate()方法创建一个可变的Exchange构建器
  ServerWebExchange.Builder exchangeBuilder = exchange.mutate();
  // 修改请求对象
  exchangeBuilder.request(modifiedRequest);
  // 构建一个新的ServerWebExchange对象
  ServerWebExchange modifiedExchange = exchangeBuilder.build();
  return chain.filter(modifiedExchange);
}
```

## 参考

Spring Cloud Gateway 官方文档：https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/

spring cloud 和 spring boot 版本兼容：https://spring.io/projects/spring-cloud#overview
