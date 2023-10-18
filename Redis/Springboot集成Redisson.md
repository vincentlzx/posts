---
title: Springboot集成Redisson
categories:
- [Redis]
tags:
- Redis
---





## redisson

Redisson是一个用于Java的开源Redis客户端，它提供了许多方便的功能和API，以简化与Redis服务器的交互和操作。

Redisson使用优化的编码协议和基于Netty的异步和非阻塞IO，以提供高性能和可伸缩性。它支持单机、主从、哨兵和集群模式的Redis部署，并提供了许多高级功能，如分布式锁、分布式集合、分布式对象、分布式发布/订阅、分布式任务调度等。

## 示例

在 springboot 项目中，使用 redisson-spring-boot-starter 替换 spring-boot-starter-data-redis 。

> springboot 版本为 2.5.12

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
<!--        <dependency>-->
<!--            <groupId>org.springframework.boot</groupId>-->
<!--            <artifactId>spring-boot-starter-data-redis</artifactId>-->
<!--        </dependency>-->
        <dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson-spring-boot-starter</artifactId>
            <version>3.22.1</version>
            <exclusions>
                <exclusion>
                    <groupId>org.redisson</groupId>
                    <artifactId>redisson-spring-data-31</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson-spring-data-25</artifactId>
            <version>3.22.1</version>
        </dependency>
<!--        <dependency>-->
<!--            <groupId>com.alibaba</groupId>-->
<!--            <artifactId>fastjson</artifactId>-->
<!--            <version>1.2.80</version>-->
<!--        </dependency>-->
        <dependency>
            <groupId>com.alibaba.fastjson2</groupId>
            <artifactId>fastjson2</artifactId>
            <version>2.0.33</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
    </dependencies>
```

redisson 可以使用常见的 springboot 配置方式，也有 redisson 自己的配置方式。一般情况下，使用 redisson-spring-boot-starter 替换 spring-boot-starter-data-redis 后是可以兼容原先根据 spring-boot-starter-data-redis 编写的代码。

```yaml
spring:
  redis:
#    cluster:
#      max-redirects: 5
#      nodes: 127.0.0.1:6379,127.0.0.1:6380,127.0.0.1:6381
#    # 在 Redis Cluster 中，数据库索引的概念被忽略。
#    database: 0
#    # 密码
#    password: null
#    # 连接redis服务器超时。
#    connect-timeout: 10000
#    # redis服务器响应超时。Redis 命令发送成功后开始倒计时。
#    timeout: 3000
#    lettuce:
#      pool:
#        # 连接池的最小空闲连接数
#        min-idle: 0
#        # 连接池的最大空闲连接数
#        max-idle: 8
#        # 连接池的最大活跃连接数
#        max-active: 8
#        # 从连接池获取连接的最大等待时间（-1表示无限等待）
#        max-wait: -1

    redisson:
      config: |
        clusterServersConfig:
          idleConnectionTimeout: 10000
          connectTimeout: 10000
          timeout: 3000
          retryAttempts: 3
          retryInterval: 1500
          failedSlaveReconnectionInterval: 3000
          failedSlaveCheckInterval: 60000
          password: null
          subscriptionsPerConnection: 5
          clientName: null
          loadBalancer: !<org.redisson.connection.balancer.RoundRobinLoadBalancer> {}
          subscriptionConnectionMinimumIdleSize: 1
          subscriptionConnectionPoolSize: 50
          slaveConnectionMinimumIdleSize: 24
          slaveConnectionPoolSize: 64
          masterConnectionMinimumIdleSize: 24
          masterConnectionPoolSize: 64
          readMode: "MASTER"
          subscriptionMode: "MASTER"
          nodeAddresses:
          - "redis://127.0.0.1:6379"
          - "redis://127.0.0.1:6380"
          - "redis://127.0.0.1:6381"
          scanInterval: 1000
          pingConnectionInterval: 0
          keepAlive: false
          tcpNoDelay: false
        threads: 16
        nettyThreads: 32
        codec: !<org.redisson.codec.Kryo5Codec> {}
        transportMode: "NIO"
```



## 参考

redisson 官方文档：https://github.com/redisson/redisson/blob/master/redisson-spring-boot-starter/README.md

项目地址：https://github.com/vincentlzx/demo/blob/main/demo-redisson/README.md