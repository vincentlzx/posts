---
title: Spring Cloud Gateway集成Sentinel
categories:
- [SpringCloud, Gateway]
tags:
- SpringCloud
- Gateway
- Sentinel
---



## Gateway整合Sentinel

从 1.6.0 版本开始，Sentinel 提供了 Spring Cloud Gateway 的适配模块，可以提供两种资源维度的限流：

- route 维度：即在 Spring 配置文件中配置的路由条目，资源名为对应的 routeId
- 自定义 API 维度：用户可以利用 Sentinel 提供的 API 来自定义一些 API 分组（分组定义参考 [sentinel-demo-spring-cloud-gateway](https://github.com/alibaba/Sentinel/tree/master/sentinel-demo/sentinel-demo-spring-cloud-gateway)）

1. 引入以下模块（以 Maven 为例）：

```xml
<!-- SpringCloud Gateway -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

<!-- SpringCloud Alibaba Sentinel -->
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>

<!-- SpringCloud Alibaba Sentinel Gateway -->
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
</dependency>

<!-- Sentinel Datasource Nacos -->
<dependency>
  <groupId>com.alibaba.csp</groupId>
  <artifactId>sentinel-datasource-nacos</artifactId>
</dependency>
```

2. 注入对应的 `SentinelGatewayFilter` 实例。

```java
@Configuration
public class GatewayConfiguration {
    @Bean
    @Order(-1)
    public SentinelGatewayFilter sentinelGatewayFilter() {
        return new SentinelGatewayFilter();
    }
}
```

这一步忽略也可以，因为自动配置类 SentinelSCGAutoConfiguration 会帮我们自动注入。部分源码如下

```java
@Bean
@Order(-1)
@ConditionalOnMissingBean
public SentinelGatewayFilter sentinelGatewayFilter() {
  logger.info("[Sentinel SpringCloudGateway] register SentinelGatewayFilter with order: {}", this.gatewayProperties.getOrder());
  return new SentinelGatewayFilter(this.gatewayProperties.getOrder());
}
```

3. 自定义 Sentinel 异常处理，注入 `SentinelGatewayBlockExceptionHandler` 实例。

```java
@Configuration
public class GatewayConfiguration {
    @Bean
    @Order(Ordered.HIGHEST_PRECEDENCE)
    public WebExceptionHandler sentinelGatewayExceptionHandler() {
        // Register the block exception handler for Spring Cloud Gateway.
        return new SentinelGatewayExceptionHandler();
    }
}

public class SentinelGatewayExceptionHandler implements WebExceptionHandler {
    @Override
    public Mono<Void> handle(ServerWebExchange exchange, Throwable ex) {
        if (exchange.getResponse().isCommitted()) {
            return Mono.error(ex);
        }
        if (!BlockException.isBlockException(ex)) {
            return Mono.error(ex);
        }
        return ServletUtils.webFluxResponseWriter(exchange.getResponse(), "系统繁忙，请稍候再试");
    }
}
```

4. 定义网关限流规则。使用 nacos 数据源方式配置规则如下：

```yaml
spring: 
  application:
    # 应用名称
    name: demo-gateway
  profiles:
    # 环境配置
    active: dev
  cloud:
    nacos:
      discovery:
        # 服务注册地址
        server-addr: liuzx.com.cn:8848
        namespace: 8fdd1e0d-583f-40e5-b7d3-bd086700b217
        group: DEFAULT_GROUP
        heart-beat-timeout: 30000
        heart-beat-interval: 10000
      # 启用nacos配置中心
      config:
        # 配置中心地址
        server-addr: liuzx.com.cn:8848
        namespace: 8fdd1e0d-583f-40e5-b7d3-bd086700b217
        group: DEFAULT_GROUP
        # 配置文件格式
        file-extension: yml
        # 共享配置
        shared-configs:
          - application-${spring.profiles.active}.${spring.cloud.nacos.config.file-extension}
      username: nacos
      password: nacos
    sentinel:
      # 取消控制台懒加载
      eager: true
      transport:
        # 控制台地址
        dashboard: liuzx.com.cn:18080
        port: 8080
        # 公网IP，使用域名不生效
        client-ip: 175.178.51.17
      # nacos配置持久化
      datasource:
        ds1:
          nacos:
            server-addr: liuzx.com.cn:8848
            namespace: 8fdd1e0d-583f-40e5-b7d3-bd086700b217
            group-id: DEFAULT_GROUP
            dataId: demo-gateway-sentinel
            data-type: json
            rule-type: gw-flow
            username: nacos
            password: nacos
```

> rule-type: gw-flow ，规则类型必须是 gw-flow ，表示网关限流规则类型。

在 nacos 配置中心编写配置文件 demo-gateway-sentinel

```json
[
    {
        "resource": "demo-fanyi",
        "count": 1,
        "intervalSec": 5,
        "grade": 1,
        "controlBehavior": 0
    }
]
```

表示对 routeId 为 demo-fanyi 的资源每 5 秒访问一次的限流。

## 参考

Sentinel 网关流量控制官方文档：https://sentinelguard.io/zh-cn/docs/api-gateway-flow-control.html

网关限流规则 `GatewayFlowRule` 的字段解释：https://github.com/alibaba/Sentinel/wiki/%E7%BD%91%E5%85%B3%E9%99%90%E6%B5%81

Demo 示例：[sentinel-demo-spring-cloud-gateway](https://github.com/alibaba/Sentinel/tree/master/sentinel-demo/sentinel-demo-spring-cloud-gateway)