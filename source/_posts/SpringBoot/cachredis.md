---
title: Spring Boot（十七）：缓存之Redis
date: 2018-09-02 17:11:03
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

EhCache更多的是在本地做缓存，redis做集群比EhCache简单轻量。

pom文件引入：

~~~java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-redis</artifactId>
</dependency>
~~~

<!-- more -->

配置文件：

~~~java
spring.cache.type=redis
~~~

缓存使用优先级：

1. 默认按照 spring boot 的加载顺序来实现
   - Generic 
   - JCache (JSR-107)
   - EhCache 2.x 
   - Hazelcast
   - Infinispan 
   - Couchbase 
   - Redis 
   - Caffeine 
   - Guava 
   - Simple  

2. 配置文件优先于默认  

通用注解：

redis缓存和EhCache缓存实现使用注解相同。redis做缓存的对象需要序列化。

所以一种实现方式是使用上一节ehcache的缓存策略和测试类测试接口等，测试结果如下：

查找：

![](cachredis\redisCache01.png)

更新（2次更新）：

![](cachredis\redisCache02.png)

redis自定义缓存管理器：

~~~java
/**
 * 自定义缓存管理器.
 *
 */
@Configuration
public class RedisCacheConfiguration extends CachingConfigurerSupport {

    @Bean
    public CacheManager cacheManager(RedisTemplate<?, ?> redisTemplate) {
        RedisCacheManager cacheManager = new RedisCacheManager(redisTemplate);
        // 设置默认的过期时间
        cacheManager.setDefaultExpiration(20);
        Map<String, Long> expires = new HashMap<String, Long>();
        // 单独设置
        expires.put("doddCache", 200L);
        cacheManager.setExpires(expires);
        return cacheManager;
    }


    /**
     * 自定义 key 的生成策略
     * 自定义 key. 此方法将会根据类名+方法名+所有参数的值生成唯一的一个 key,
     * 即使@Cacheable 中的 value 属性一样， key 也会不一样。
     */
    @Override
    public KeyGenerator keyGenerator() {
        return new KeyGenerator() {
            @Override
            public Object generate(Object o, Method method, Object... objects) {
                StringBuilder sb = new StringBuilder();
                sb.append(o.getClass().getName());
                sb.append(method.getName());
                for (Object obj : objects) {
                    sb.append(obj.toString());
                }
                return sb.toString();
            }
        };
    }
}
~~~



