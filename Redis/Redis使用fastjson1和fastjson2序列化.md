---
title: Redis使用fastjson1和fastjson2序列化
categories:
- [Redis]
tags:
- Redis
---



## redis使用fastjson1.x序列化

1. 使用 fastjson1.x 版本

```xml
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>fastjson</artifactId>
  <version>1.2.80</version>
</dependency>
```

2. 实现 redis 序列化接口 RedisSerializer<T>

```java
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.ParserConfig;
import com.alibaba.fastjson.serializer.SerializerFeature;
import org.springframework.data.redis.serializer.RedisSerializer;
import org.springframework.data.redis.serializer.SerializationException;

import java.nio.charset.Charset;
import java.nio.charset.StandardCharsets;

public class FastJson2JsonRedisSerializer implements RedisSerializer<Object>
{
    public static final Charset DEFAULT_CHARSET = StandardCharsets.UTF_8;

    public FastJson2JsonRedisSerializer()
    {
        super();
    }

    static
    {
        ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
    }

    @Override
    public byte[] serialize(Object t) throws SerializationException
    {
        if (t == null)
        {
            return new byte[0];
        }
        return JSON.toJSONString(t, SerializerFeature.WriteClassName).getBytes(DEFAULT_CHARSET);
    }

    @Override
    public Object deserialize(byte[] bytes) throws SerializationException
    {
        if (bytes == null || bytes.length <= 0)
        {
            return null;
        }
        String str = new String(bytes, DEFAULT_CHARSET);
        return com.alibaba.fastjson.JSON.parseObject(str, Object.class);
    }
}
```

3. 创建 redis 配置文件，向 Spring 容器注入自定义 redisTemplate

```java
@Configuration
@EnableCaching
public class RedisConfig extends CachingConfigurerSupport {
    @Bean
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory connectionFactory)
    {
        RedisTemplate<Object, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);

        FastJson2JsonRedisSerializer serializer = new FastJson2JsonRedisSerializer();

        // 使用StringRedisSerializer来序列化和反序列化redis的key值
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(serializer);

        // Hash的key也采用StringRedisSerializer的序列化方式
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(serializer);

        template.afterPropertiesSet();
        return template;
    }
}
```



## 升级fastjson2不兼容

fastjson1.x 升级到 fastjson2.x ，反序列化对象出现 autoType 异常。

```java
com.alibaba.fastjson.JSONException: safeMode not support autoType : com.luwei.domain.TaskResult
```

> FASTJSON支持AutoType功能，这个功能在序列化的JSON字符串中带上类型信息，在反序列化时，不需要传入类型，实现自动类型识别。

### 解决方案

根据 fastjson2 官方文档修改序列化方式。注意引入的是 fastjson2 的 com.alibaba.fastjson2.JSON ，而不是 fastjson 的 com.alibaba.fastjson.JSON

> 参考：https://github.com/alibaba/fastjson2/wiki/fastjson2_autotype_cn

```java
import com.alibaba.fastjson.parser.ParserConfig;
import com.alibaba.fastjson2.JSON;
import com.alibaba.fastjson2.JSONReader;
import com.alibaba.fastjson2.JSONWriter;
import org.springframework.data.redis.serializer.RedisSerializer;
import org.springframework.data.redis.serializer.SerializationException;

import java.nio.charset.Charset;
import java.nio.charset.StandardCharsets;

public class FastJson2JsonRedisSerializer implements RedisSerializer<Object>
{
    public static final Charset DEFAULT_CHARSET = StandardCharsets.UTF_8;

    public FastJson2JsonRedisSerializer()
    {
        super();
    }

    static
    {
        ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
    }

    @Override
    public byte[] serialize(Object t) throws SerializationException
    {
        if (t == null)
        {
            return new byte[0];
        }
        return JSON.toJSONString(t, JSONWriter.Feature.WriteClassName).getBytes(DEFAULT_CHARSET);
    }

    @Override
    public Object deserialize(byte[] bytes) throws SerializationException
    {
        if (bytes == null || bytes.length <= 0)
        {
            return null;
        }
        String str = new String(bytes, DEFAULT_CHARSET);
        return JSON.parseObject(str, Object.class, JSONReader.Feature.SupportAutoType);
    }
}
```

