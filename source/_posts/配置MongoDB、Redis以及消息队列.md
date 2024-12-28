---
title: 配置MongoDB、Redis以及消息队列 5
date: 2024-12-23 22:13:29
tags:
---

## 配置MongoDB

在 `application.yml` 文件中，配置MongoDB连接。因为MongoDB自带数据库连接池，所以我们不需要在Java项目中重复配置连接池。

```yaml
spring:
......
  data:
    mongodb:
      host: localhost
      port: 27017
      database: his
      authentication-database: admin
      username: admin
      password: abc123456
```

## 配置Redis

在 `application.yml` 文件中，配置Redis连接。

```yaml
spring:
......
  redis:
    host: localhost
    port: 6379
    password: abc123456
```

### 避免RedisTemplate保存乱码数据

因为SpringBoot Data中默认的RedisTemplate存在序列化机制的问题，向Redis里面保存Hash类型数据通常是乱码的，为了解决这个问题，我们需要自己定义配置类，修改RedisTemplate使用的序列化机制。

在`com.example.his.api.config`包中，创建 `RedisTemplateconfig` 类

```java
package com.jiang.his.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisTemplateConfig {
    @Bean
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<Object, Object> template = new RedisTemplate<>();
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
}
```

## 配置RabbitMQ消息队列

在 application.yml 文件中，我们填上消息队列的配置信息。Java项目启动后，SpringBoot会自动接收RabbitMQ中的消息。

```yaml
spring:
......
  rabbitmq:
    host: localhost
    port: 5672
    username: admin
    password: abc123456
```
