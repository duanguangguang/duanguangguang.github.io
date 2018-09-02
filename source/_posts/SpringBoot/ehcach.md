---
title: Spring Boot（十六）：缓存之EhCach
date: 2018-09-02 17:11:03
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

Spring boot 支持的缓存： 

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

<!-- more -->

pom文件引入：

~~~java
<dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<!-- Ehcach -->
<dependency>
     <groupId>net.sf.ehcache</groupId>
     <artifactId>ehcache</artifactId>
</dependency>
~~~

配置文件：

~~~java
spring.cache.type=ehcache
spring.cache.ehcache.config=classpath:config/ehcache.xml
~~~

ehcache.xml：

~~~html
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="ehcache.xsd">
    <cache name="doddCache"
           eternal="false"
           maxEntriesLocalHeap="0"
           timeToIdleSeconds="200"></cache>
    <!-- eternal： true表示对象永不过期，此时会忽略timeToIdleSeconds和
                   timeToLiveSeconds属性，默认为false -->
    <!-- maxEntriesLocalHeap：堆内存中最大缓存对象数， 0表示没有限制 -->
    <!-- timeToIdleSeconds： 设定允许对象处于空闲状态的最长时间，以秒为单位。
					当对象自从最近一次被访问后，如果处于空闲状态的时间超过了timeToIdleSeconds
					属性值，这个对象就会过期， EHCache将把它从缓存中清空。只有当eternal属性为						false，该属性才有效。如果该属性值为0，则表示对象可以无限期地处于空闲状态 -->
</ehcache>
~~~

接口：

~~~java
public interface EhCache {
    /**
     * 查询
     */
    JpaUser selectById(Integer id);
    /**
     * 更新
     */
    JpaUser updateById(JpaUser jpaUser);
    /**
     * 删除
     */
    String deleteById(Integer id);
}
~~~

实现类：

~~~java
/**
 * @CacheConfig： 缓存配置
 */
@CacheConfig(cacheNames = "doddCache")
@Repository
public class EhCacheImpl implements EhCache {

    @Autowired
    private JpaUserDao jpaUserDao;

    /**
     * @Cacheable： 应用到读取数据的方法上，即可缓存的方法，如查找方法：先从缓存中读取，如果没有再调
     *              用方法获取数据，然后把数据添加到缓存中。 适用于查找
     */
    @Cacheable(key = "#p0")
    @Override
    public JpaUser selectById(Integer id) {
        System.out.println("查询功能，缓存找不到，直接读库, id=" +  id);
        return jpaUserDao.findOne(id);
    }

    /**
     * @CachePut： 主要针对方法配置，能够根据方法的请求参数对其结果进行缓存，和 @Cacheable 不同的
     *             是，它每次都会触发真实方法的调用。 适用于更新和插入
     */
    @CachePut(key = "#p0")
    @Override
    public JpaUser updateById(JpaUser jpaUser) {
        System.out.println("更新功能，更新缓存，直接写库, id=" + jpaUser);
        return jpaUserDao.save(jpaUser);
    }

    /**
     * @CacheEvict： 主要针对方法配置，能够根据一定的条件对缓存进行清空。 适用于删除
     */
    @CacheEvict(key = "#p0")
    @Override
    public String deleteById(Integer id) {
        System.out.println("删除功能，删除缓存，直接写库, id=" + id);
        return "清空缓存成功";
    }

}
~~~

启动类启用缓存注解：@EnableCaching 

测试缓存：

~~~java
@RestController
@RequestMapping("/ehcache")
public class EhCacheController {
    @Autowired
    private EhCache ehCache;

    @RequestMapping(value = "/select", method = RequestMethod.GET)
    public JpaUser get(@RequestParam(defaultValue = "2") Integer id) {
        return ehCache.selectById(id);
    }

    @RequestMapping(value = "/update", method = RequestMethod.GET)
    public JpaUser update(@RequestParam(defaultValue = "2") Integer id) {
        JpaUser bean = ehCache.selectById(id);
        bean.setUserName("dodd22");
        bean.setCreateTime(new Date());
        ehCache.updateById(bean);
        return bean;
    }

    @RequestMapping(value = "/del", method = RequestMethod.GET)
    public String del(@RequestParam(defaultValue = "1") Integer id) {
        return ehCache.deleteById(id);
    }
}
~~~

缓存测试结果：

查询：

![](ehcach\eh01.png)

更新：

![](ehcach\eh02.png)

