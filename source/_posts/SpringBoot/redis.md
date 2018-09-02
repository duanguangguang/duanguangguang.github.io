---
title: Spring Boot（十四）：NoSql数据库redis
date: 2018-09-02 13:59:44
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

pom文件引入：

~~~java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
~~~

<!-- more -->

添加配置：

~~~java
#redis不配置会读取默认配置
spring.redis.host=localhost
spring.redis.port=6379
#spring.redis.password=123456
#spring.redis.database=0
#spring.redis.pool.max-active=8
#spring.redis.pool.max-idle=8
#spring.redis.pool.max-wait=-1
#spring.redis.pool.min-idle=0
#spring.redis.timeout=0
~~~

实现类：

~~~java
@Component
public class RedisComponent {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    public void set(String key, String value) {
        ValueOperations<String, String> ops = this.stringRedisTemplate.opsForValue();
        if (!this.stringRedisTemplate.hasKey(key)) {
            ops.set(key, value);
            System.out.println("set key success");
        } else {
            // 存在则打印之前的 value 值
            System.out.println("this key = " + ops.get(key));
        }
    }
    public String get(String key) {
        return this.stringRedisTemplate.opsForValue().get(key);
    }
    public void del(String key) {
        this.stringRedisTemplate.delete(key);
    }
}
~~~

测试类：

~~~java
@Autowired
private RedisComponent redisComponent;
@Test
public void set() {
    redisComponent.set("dodd", "hello world");
}
@Test
public void get() {
    System.out.println(redisComponent.get("dodd"));
}
@Test
public void del() {
    redisComponent.del("dodd");
}
~~~



